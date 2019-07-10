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

Также AOTMapper можно установить через nuget:
```
Install-Package AOTMapper
```

Я долго думал как это можно сделать по другому и в конце-концов сошелся на мысли, что это не так уж и плохо, поскольку это решает некоторые неудобства, которые мучили меня при использовании ``` AutoMapper ```. 

Во первых, мы получаем разные методы расширения для разных типов, в следствии чего, для него абстрактного типа ``` User ``` мы можем очень легко при помощи IntelliSense-а узнать какие мапинги уже реализованы без необходимости искать тот файл, где прописаны наши мапинги. Достаточно лишь посмотреть какие методы расширения уже есть.

Во вторых, в рантайме это просто метод расширение и таким образом мы избегаем любых накладных расходов связанных с вызовом нашего мапера. Я понимаю что разработчики ``` AutoMapper ``` потратили много усилий на оптимизацию вызова, но дополнительные затраты всеравно есть. Мой небольшой бенчмарк показал что в среднем это 140-150ns на вызов, без учета времени на инициализацию. Сам бенчмарк можно посмотреть в репозиторий, а результаты замеров ниже.

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

Все что он делает это проверяет тот ли это метод который нам нужен, извлекает тип из сущности на которой вызван ``` MapTo<> ``` вместе из первым параметром обобщенного метода и генерит диагностическое сообщение. 

Оно уже в свою очередь будет обработано внутри ``` AOTMapperCodeFixProvider```. Здесь мы достает информацию о типах над которыми будем запускать кодогенерацию. Потом заменяем вызов ``` MapTo<> ``` на конкретную реализацию. После чего вызываем ``` AOTMapperGenerator ``` который сгенерит нам файл с методом расширением.

В коде это имеет следующий вид: 

``` cs 
 private async Task<Document> Handle(Diagnostic diagnostic, CodeFixContext context)
{
    var fromTypeName = diagnostic.Properties["fromType"];
    var toTypeName = diagnostic.Properties["toType"];
    var document = context.Document;

    var semanticModel = await document.GetSemanticModelAsync();
    
    var root = await diagnostic.Location.SourceTree.GetRootAsync();
    var call = root.FindNode(diagnostic.Location.SourceSpan);
    root = root.ReplaceNode(call, SyntaxFactory.IdentifierName($"MapTo{toTypeName.Split('.').Last()}"));

    var pairs = ImmutableDictionary<string, string>.Empty
        .Add(fromTypeName, toTypeName);

    var generator = new AOTMapperGenerator(document.Project, semanticModel.Compilation);
    generator.GenerateMappers(pairs, new[] { "AOTMapper", "Mappers" });

    var newProject = generator.Project;
    var documentInNewProject = newProject.GetDocument(document.Id);

    return documentInNewProject.WithSyntaxRoot(root);
}
```

Сам ``` AOTMapperGenerator ``` изменяет входящий проект добаляя к нему реализацию мапинга между типами.
Сделано это следующим образом:
``` cs
public void GenerateMappers(ImmutableDictionary<string, string> values, string[] outputNamespace)
{
    foreach (var value in values)
    {
        var fromSymbol = this.Compilation.GetTypeByMetadataName(value.Key);
        var toSymbol = this.Compilation.GetTypeByMetadataName(value.Value);
        var fromSymbolName = fromSymbol.ToDisplayString().Replace(".", "");
        var toSymbolName = toSymbol.ToDisplayString().Replace(".", "");
        var fileName = $"{fromSymbolName}_To_{toSymbolName}";

        var source = this.GenerateMapper(fromSymbol, toSymbol, fileName);

        
        this.Project = this.Project
            .AddDocument($"{fileName}.cs", source)
            .WithFolders(outputNamespace)
            .Project;
    }
}

private string GenerateMapper(INamedTypeSymbol fromSymbol, INamedTypeSymbol toSymbol, string fileName)
{
    var fromProperties = fromSymbol.GetAllMembers()
        .OfType<IPropertySymbol>()
        .Where(o => (o.DeclaredAccessibility & Accessibility.Public) > 0)
        .ToDictionary(o => o.Name, o => o.Type);

    var toProperties = toSymbol.GetAllMembers()
        .OfType<IPropertySymbol>()
        .Where(o => (o.DeclaredAccessibility & Accessibility.Public) > 0)
        .ToDictionary(o => o.Name, o => o.Type);

        return $@"
public static class {fileName}Extentions 
{{
    public static {toSymbol.ToDisplayString()} MapTo{toSymbol.ToDisplayString().Split('.').Last()}(this {fromSymbol.ToDisplayString()} input)
    {{
        var output = new {toSymbol.ToDisplayString()}();
{ toProperties
    .Where(o => fromProperties.TryGetValue(o.Key, out var type) && type == o.Value)
    .Select(o => $"        output.{o.Key} = input.{o.Key};" )
    .JoinWithNewLine()
}
{ toProperties
    .Where(o => !fromProperties.TryGetValue(o.Key, out var type) || type != o.Value)
    .Select(o => $"        output.{o.Key} = ; // missing property")
    .JoinWithNewLine()
}
        return output;
    }}
}}
";
}
```

Полный код проекта можно посмотреть [тут](https://github.com/byme8/AOTMapper).


Серед текущих недоработок которые я заметил можно выделить то, что из коробки нельзя сгенерить две реализации для одиинаковых типов, но всегда можно вручную переименовать уже сгенереный файл/метод это никак не повлияет на работоспособность. Кроме того, сейчас нет проверки изменился ли тип после последней генерации. У меня есть идея как это можно реализовать, но меня слущает что это может существенно увеличить нагрузку, а проверить это на деле руки пока не дошли.
