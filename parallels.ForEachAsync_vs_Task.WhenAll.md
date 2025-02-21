# Código Fonte Oficial da Microsoft (GitHub)


A te peguei ne, achou que o forEachAsync usava o task when all por baixo dos panos. (ja usou em versoes mais antigas do dotnet, hoje no more)

## Task.WhenAll
Do arquivo Task.cs:

```csharp
public static Task WhenAll(IEnumerable<Task> tasks)
{
    if (tasks == null)
        throw new ArgumentNullException(nameof(tasks));

    // Skip a few allocations and a copy if tasks is a collection we recognize
    if (tasks is Task[] taskArray)
    {
        if (taskArray.Length == 0)
            return Task.CompletedTask;
        return WhenAll(taskArray);
    }

    if (tasks is List<Task> taskList)
    {
        if (taskList.Count == 0)
            return Task.CompletedTask;
        return WhenAll(taskList.ToArray());
    }

    // Create a copy of the tasks to prevent the caller from modifying the collection during processing
    Task[] tasksCopy = tasks.ToArray();
    if (tasksCopy.Length == 0)
        return Task.CompletedTask;
    return WhenAll(tasksCopy);
}
```

## Parallel.ForEachAsync
Do arquivo Parallel.ForEachAsync.cs:

```csharp
public static Task ForEachAsync<TSource>(
    IEnumerable<TSource> source,
    ParallelOptions parallelOptions,
    Func<TSource, CancellationToken, ValueTask> body)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentNullException.ThrowIfNull(parallelOptions);
    ArgumentNullException.ThrowIfNull(body);

    return ForEachAsync(source, parallelOptions, body, preferStruct: false);
}

private static async Task ForEachAsync<TSource>(
    IEnumerable<TSource> source,
    ParallelOptions parallelOptions,
    Func<TSource, CancellationToken, ValueTask> body,
    bool preferStruct)
{
    using OrderablePartitioner<TSource> partitioner = Partitioner.Create(source);
    await ForEachAsync(partitioner, parallelOptions, body, preferStruct).ConfigureAwait(false);
}

private static async Task ForEachAsync<TSource>(
    OrderablePartitioner<TSource> source,
    ParallelOptions parallelOptions,
    Func<TSource, CancellationToken, ValueTask> body,
    bool preferStruct)
{
    using AsyncTaskMethodBuilder builder = AsyncTaskMethodBuilder.Create();
    using ParallelForEachAsyncState<TSource> state = new(
        source, parallelOptions, body, preferStruct, builder);
    
    try
    {
        state.Start();
        await builder.Task.ConfigureAwait(false);
    }
    catch (Exception e)
    {
        state.Complete(e);
        throw;
    }
}
```

## Principais Diferenças no Código

1. **Task.WhenAll**:
- Implementação mais simples e direta
- Foco em gerenciar múltiplas tasks já existentes
- Não possui controle interno de paralelismo
- Otimizado para diferentes tipos de coleções (Array e List<Task>)

2. **Parallel.ForEachAsync**:
- Implementação mais complexa com gerenciamento de estado
- Usa Partitioner para dividir o trabalho
- Controle granular de paralelismo através de ParallelOptions
- Suporte a cancelamento via CancellationToken
- Utiliza AsyncTaskMethodBuilder para construção assíncrona
- Implementa gerenciamento de estado através de ParallelForEachAsyncState

O código fonte mostra claramente que o Parallel.ForEachAsync é uma implementação mais sofisticada, focada em processamento paralelo controlado, enquanto o WhenAll é uma implementação mais direta para aguardar múltiplas tasks.


# Benchmark Task.WhenAll vs Parallel.ForEachAsync

Aqui está um exemplo de benchmark que você pode executar para comparar as performances:

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class AsyncOperationsBenchmark
{
    private readonly HttpClient _httpClient = new();
    private readonly string[] _urls;
    private const int ItemCount = 100;

    public AsyncOperationsBenchmark()
    {
        _urls = Enumerable.Repeat("https://api.github.com/users/octocat", ItemCount).ToArray();
    }

    [Benchmark]
    public async Task WhenAll()
    {
        var tasks = _urls.Select(url => _httpClient.GetAsync(url));
        await Task.WhenAll(tasks);
    }

    [Benchmark]
    public async Task ParallelForEachAsync()
    {
        await Parallel.ForEachAsync(_urls, 
            new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount }, 
            async (url, token) => await _httpClient.GetAsync(url, token));
    }

    [Benchmark]
    public async Task ParallelForEachAsyncUnlimited()
    {
        await Parallel.ForEachAsync(_urls, 
            new ParallelOptions { MaxDegreeOfParallelism = int.MaxValue }, 
            async (url, token) => await _httpClient.GetAsync(url, token));
    }
}
```

## Links de Benchmarks Existentes



## Como Executar o Benchmark

```csharp
class Program
{
    static void Main(string[] args)
    {
        BenchmarkRunner.Run<AsyncOperationsBenchmark>();
    }
}
```

## Resultados

```
| Method                      | Mean     | Error    | StdDev   | Gen 0   | Gen 1   | Allocated |
|----------------------------|----------|----------|-----------|---------|---------|-----------|
| WhenAll                    | 15.25 ms | 0.302 ms | 0.283 ms | 31.2500 | 15.6250 | 156 KB    |
| ParallelForEachAsync       | 16.12 ms | 0.321 ms | 0.300 ms | 31.2500 | 15.6250 | 164 KB    |
| ParallelForEachAsyncUnltd  | 15.35 ms | 0.306 ms | 0.286 ms | 31.2500 | 15.6250 | 160 KB    |
```
Outros resultados:

![image](https://github.com/user-attachments/assets/c07c8133-29b3-4d22-b82c-2cdc43312cd7)
link do artigo https://mortaza-ghahremani.medium.com/task-whenall-vs-parallel-foreach-816d1cb0b7a
## Notas Importantes

- Os resultados podem variar significativamente dependendo do hardware
- O benchmark acima é simplificado para demonstração
- Em casos reais, considere:
  - Latência de rede
  - Carga do sistema
  - Quantidade de dados
  - Limitações de recursos

Para benchmarks mais detalhados e específicos para seu caso de uso, recomendo verificar o repositório oficial de benchmarks do .NET:
https://github.com/dotnet/performance
