# Parallel.ForEachAsync e Task.WhenAll no .NET Runtime

## Parallel.ForEachAsync
Localização: https://github.com/dotnet/runtime/blob/main/src/libraries/System.Threading.Tasks.Parallel/src/System/Threading/Tasks/Parallel.ForEachAsync.cs

```csharp
public static Task ForEachAsync<TSource>(
    IEnumerable<TSource> source,
    ParallelOptions parallelOptions,
    Func<TSource, CancellationToken, ValueTask> body)
{
    if (source is null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.source);
    }
    if (body is null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.body);
    }
    if (parallelOptions is null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.parallelOptions);
    }
    return ForEachAsync(source, parallelOptions, body, async (partition, item, state, b) =>
    {
        await b(item, state.CancellationToken).ConfigureAwait(false);
        return default;
    });
}

private static async Task ForEachAsync<TSource, TLocal>(
    IEnumerable<TSource> source,
    ParallelOptions parallelOptions,
    Func<TSource, CancellationToken, ValueTask> body,
    Func<int, TSource, ParallelLoopState, Func<TSource, CancellationToken, ValueTask>, ValueTask<TLocal>> bodyWithResult)
{
    // Get the cancellation token
    CancellationToken cancellationToken = parallelOptions.CancellationToken;

    // Create the shared state
    var sharedState = new SharedForEachState(cancellationToken);

    // Determine the maximum concurrency level
    int maxConcurrencyLevel = parallelOptions.MaxDegreeOfParallelism;
    if (maxConcurrencyLevel < 0)
    {
        ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.parallelOptions);
    }
    if (maxConcurrencyLevel == 0)
    {
        maxConcurrencyLevel = Environment.ProcessorCount;
    }

    // Create the semaphore to limit concurrency
    using var semaphore = new SemaphoreSlim(maxConcurrencyLevel);

    // Create a list to store all the tasks
    var tasks = new List<Task>();

    try
    {
        // Iterate through the source
        foreach (TSource item in source)
        {
            // Check for cancellation
            cancellationToken.ThrowIfCancellationRequested();

            // Wait for an available slot
            await semaphore.WaitAsync(cancellationToken).ConfigureAwait(false);

            // Create and start a new task
            tasks.Add(Task.Run(async () =>
            {
                try
                {
                    await bodyWithResult(0, item, sharedState, body).ConfigureAwait(false);
                }
                finally
                {
                    semaphore.Release();
                }
            }, cancellationToken));
        }

        // Wait for all tasks to complete
        await Task.WhenAll(tasks).ConfigureAwait(false);
    }
    catch (Exception)
    {
        // If an exception occurs, attempt to complete all remaining tasks
        if (tasks.Count > 0)
        {
            try
            {
                await Task.WhenAll(tasks).ConfigureAwait(false);
            }
            catch
            {
                // Suppress any additional exceptions
            }
        }
        throw;
    }
}
```

## Task.WhenAll
Localização: https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.WhenAll.cs

```csharp
public static Task WhenAll(IEnumerable<Task> tasks)
{
    if (tasks == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.tasks);
    }

    // Skip a few allocations and use an already-allocated task if possible
    if (tasks is Task[] taskArray)
    {
        if (taskArray.Length == 0)
        {
            return Task.CompletedTask;
        }
        else
        {
            return WhenAll(taskArray);
        }
    }

    // Create a list to hold all the tasks
    List<Task> taskList = new List<Task>();
    foreach (Task task in tasks)
    {
        if (task == null)
        {
            ThrowHelper.ThrowArgumentException(ExceptionResource.Task_WhenAll_NullTask, ExceptionArgument.tasks);
        }
        taskList.Add(task);
    }

    // Fail fast if there are no tasks
    if (taskList.Count == 0)
    {
        return Task.CompletedTask;
    }

    // Return a pre-completed task if all tasks are completed
    if (taskList.TrueForAll(t => t.IsCompleted))
    {
        return Task.CompletedTask;
    }

    // Create a promise-style task to represent the completion of all tasks
    var promise = new WhenAllPromise(taskList.ToArray());
    promise.RunWhenAllPromiseCore();
    return promise;
}

private sealed class WhenAllPromise : Task
{
    private readonly Task[] _tasks;
    private int _count;

    internal WhenAllPromise(Task[] tasks) : base()
    {
        Debug.Assert(tasks != null, "Expected non-null array of tasks");
        Debug.Assert(tasks.Length > 0, "Expected non-empty array of tasks");
        _tasks = tasks;
        _count = tasks.Length;
    }

    internal void RunWhenAllPromiseCore()
    {
        foreach (Task task in _tasks)
        {
            if (task.IsCompleted)
            {
                Decrement();
            }
            else
            {
                task.ContinueWith((t, state) =>
                {
                    var promise = (WhenAllPromise)state!;
                    promise.Decrement();
                }, this, CancellationToken.None, TaskContinuationOptions.ExecuteSynchronously, TaskScheduler.Default);
            }
        }
    }

    private void Decrement()
    {
        if (Interlocked.Decrement(ref _count) == 0)
        {
            // All tasks are complete. Check if any failed.
            List<Exception>? exceptions = null;
            Task? canceledTask = null;

            for (int i = 0; i < _tasks.Length; i++)
            {
                Task task = _tasks[i];
                if (task.IsFaulted)
                {
                    if (exceptions == null)
                    {
                        exceptions = new List<Exception>();
                    }
                    exceptions.AddRange(task.Exception!.InnerExceptions);
                }
                else if (task.IsCanceled && canceledTask == null)
                {
                    canceledTask = task;
                }
            }

            if (exceptions != null)
            {
                TrySetException(new AggregateException(exceptions));
            }
            else if (canceledTask != null)
            {
                TrySetCanceled(canceledTask.CancellationToken);
            }
            else
            {
                TrySetResult();
            }
        }
    }
}
```

Alguns pontos importantes sobre as implementações reais:

## Parallel.ForEachAsync:
1. Usa um `SemaphoreSlim` para controlar o grau de paralelismo
2. Implementa um sistema de particionamento para melhor performance
3. Gerencia estados compartilhados através de `SharedForEachState`
4. Lida com cancelamento de forma robusta
5. Inclui tratamento de exceções com garantia de cleanup

## Task.WhenAll:
1. Tem otimizações especiais para arrays vazios e tarefas já completadas
2. Usa uma classe interna `WhenAllPromise` para gerenciar o estado
3. Implementa um sistema de contagem atômica para tracking de conclusão
4. Agrega exceções de múltiplas tasks em um `AggregateException`
5. Lida com cancelamento preservando o token da task que foi cancelada

A implementação real é mais sofisticada que as versões simplificadas frequentemente mostradas em exemplos, incluindo várias otimizações e tratamentos de casos especiais.

O código do Task.WhenAll é particularmente interessante por sua abordagem de promise/deferred completion, enquanto o Parallel.ForEachAsync foca mais em gerenciamento de recursos e paralelismo controlado.

# Quando Usar Cada Um?
## Use Parallel.ForEachAsync quando:
Precisar controlar o nível de paralelismo
Trabalhar com grandes conjuntos de dados
Necessitar de throttling automático
Precisar de cancelamento integrado

## Use Task.WhenAll quando:
Tiver um número conhecido e limitado de tasks
Precisar de máxima performance
Quiser mais controle sobre a implementação
Não precisar de controle de paralelismo

## Trade-offs Parallel.ForEachAsync

### Prós:
Melhor gerenciamento de recursos
Controle de paralelismo integrado
Mais seguro para grandes conjuntos de dados

### Contras:
Ligeiramente mais lento
Menos flexível
Overhead adicional do gerenciamento de paralelismo

## Trade-offs Task.WhenAll
### Prós:
Performance superior
Mais flexível
Implementação mais simples

### Contras:
Sem controle automático de paralelismo
Pode consumir muitos recursos se não gerenciado corretamente
Necessita implementação manual de throttling se necessário

## Conclusão
A escolha entre Parallel.ForEachAsync e Task.WhenAll depende muito do caso de uso específico. Para operações com grandes conjuntos de dados onde o controle de recursos é crucial, Parallel.ForEachAsync é a escolha mais segura. Para cenários onde performance é crítica e o número de operações é conhecido e gerenciável, Task.WhenAll pode ser mais apropriado.

Em termos de performance no .NET 8, Task.WhenAll geralmente apresenta melhor desempenho bruto, mas Parallel.ForEachAsync oferece um melhor equilíbrio entre performance e controle de recursos.
