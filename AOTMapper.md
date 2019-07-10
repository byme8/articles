В прошлой [статье](https://habr.com/ru/post/455952) я описал как можно организовать кодогенерацию при помощи Roslyn. Тогдашней задачей было продемонстрировать как это вообще выглядит. Сейчас я хочу реализовать то, что будет иметь реальное применение. 

И так, кому интересно посмотреть на то как можно сделать библиотеку на подобие [AutoMapper](https://github.com/AutoMapper/AutoMapper) прошу под кат.

 <cut/>

 Первым делом, думаю стоит описать то, как будет работать мой Ahead of Time Mapper(AOTMapper). Точкой входа нашего мапера будет служить обобщенный метод расширение(generic extention method)``` MapTo<> ```. Анализатор будет искать его и предлагать реализовать метод расширение ``` MapToSomeType ```, где ``` SomeType ``` это тип который передан в ```MapTo<> ```.

 Сгенерённый ``` MapToSomeType ``` может иметь следующий вид:

 ``` cs 
public static AOTMapper.Benchmark.Data.User MapToUser(this AOTMapper.Benchmark.Data.UserEntity input)
{
    var output = new AOTMapper.Benchmark.Data.User();
    output.FirstName = input.FirstName;
    output.LastName = input.LastName;
    output.Name = ; // missing property

    return output;
}
 ```
Как видно из этого примера, все свойства с одинаковыми именами и типами присваиваются автоматически. В свою очередь, те для которых совпадения не обнаружены продолжают "висеть" создавая ошибку компиляции и разработчик должен как-то их обработать.

Например, вот так:
``` cs
public static AOTMapper.Benchmark.Data.User MapToUser(this AOTMapper.Benchmark.Data.UserEntity input)
{
    var output = new AOTMapper.Benchmark.Data.User();
    output.FirstName = input.FirstName;
    output.LastName = input.LastName;
    output.Name = $"{input.FirstName} {input.LastName}";
    
    return output;
}
```

Во время генерации ``` MapToSomeType ``` место вызова ``` MapTo<SomeType> ``` будет заменено на ``` MapToSomeType ```. 

Как это работает в движении можно посмотреть тут:

<oembed> https://youtu.be/BCznYk2n3II </oembed>

Я долго думал как это можно сделать по другому и в конце-концов сошелся на мысли, что это не так уж и плохо, поскольку это решает некоторые неудобства, которые мучили меня при использовании ``` AutoMapper ```. 

Во первых, мы получаем разные методы расширения для разных типов, в следствии чего, для него абстрактного типа ``` User ``` мы можем очень легко при помощи IntelliSense-а узнать какие мапинги уже реализованы без необходимости искать тот файл, где прописаны наши мапинги. Достаточно лишь посмотреть какие методы расширения уже есть.

Во вторых, в рантайме это просто метод расширение и таким образом мы избегаем любых накладных расходов связанных с вызовом нашего мапера. Я понимаю что разработчики ``` AutoMapper ``` потратили много усилий на оптимизацию вызова, но дополнительные затраты всеравно есть. Мой небольшой бенчмарк показал что в среднем это 140-150ns на вызов, без учета времени на инициализацию. Сам бенчмарк можно посмотреть в репозитории, а результаты замеров ниже.

|                 Method |      Mean |     Error |    StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|----------------------- |----------:|----------:|----------:|-------:|------:|------:|----------:|
| AutoMapperToUserEntity | 151.84 ns | 1.9952 ns | 1.8663 ns | 0.0253 |     - |     - |      80 B |
|  AOTMapperToUserEntity |  10.41 ns | 0.2009 ns | 0.1879 ns | 0.0152 |     - |     - |      48 B |
|       AutoMapperToUser | 197.51 ns | 2.9225 ns | 2.5907 ns | 0.0787 |     - |     - |     248 B |
|        AOTMapperToUser |  46.46 ns | 0.3530 ns | 0.3129 ns | 0.0686 |     - |     - |     216 B |


Кроме того к преимуществам данного мапера можно отнести то, что он вообще не требует времени на инициализацю при запуске приложения, что может быть полезным в больших приложениях.


Сам анализатор имеет следующий вид(упуская обвязочный код):
``` cs
private void Handle(OperationAnalysisContext context)
{
    var syntax = context.Operation.Syntax;
    if (syntax is InvocationExpressionSyntax invocationSytax &&
        invocationSytax.Expression is MemberAccessExpressionSyntax memberAccessSyntax &&
        syntax.DescendantNodes().OfType<GenericNameSyntax>().FirstOrDefault() is GenericNameSyntax genericNameSyntax &&
        genericNameSyntax.Identifier.ValueText == "MapTo")
    {
        var semanticModel = context.Compilation.GetSemanticModel(syntax.SyntaxTree);
        var methodInformation = semanticModel.GetSymbolInfo(genericNameSyntax);
        if (methodInformation.Symbol.ContainingAssembly.Name != CoreAssemblyName)
        {
            return;
        }

        var fromTypeInfo = semanticModel.GetTypeInfo(memberAccessSyntax.Expression);
        var fromTypeName = fromTypeInfo.Type.ToDisplayString();

        var typeSyntax = genericNameSyntax.TypeArgumentList.Arguments.First();
        var toTypeInfo = semanticModel.GetTypeInfo(typeSyntax);
        var toTypeName = toTypeInfo.Type.ToDisplayString();

        var properties = ImmutableDictionary<string, string>.Empty
            .Add("fromType", fromTypeName)
            .Add("toType", toTypeName);

        context.ReportDiagnostic(Diagnostic.Create(AOTMapperIsNotReadyDescriptor, genericNameSyntax.GetLocation(), properties));
    }
}
```

Все что он делает это проверяет тот ли это метод который нам нужен, извлекает тип из сущности на которой вызван ``` MapTo<> ``` вместе из первым параметром обобщенного метода и генерит диагностическое сообщение. Оно уже в свою очередь будет обработано внутри ``` AOTMapperCodeFixProvider```, а именно, заменен вызова ``` MapTo<> ``` на конкретную реализацию и сгенерен файл с методом расширением.

Весь код можно посмотреть [тут](https://github.com/byme8/AOTMapper).
Сам AOTMapper можно установить через nuget:
```
Install-Package AOTMapper
```
