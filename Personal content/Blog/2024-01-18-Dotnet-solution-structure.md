Большинство инструментов, которые Microsoft делают для dotnet можно назвать "It's just work". Но иногда стандартной конфигурации недостаточно и приходиться более подробно разбираться в проблеме. Этот пост как раз о том, как стандартного процесса сборки солюшена оказалось недостаточно и пришлось его пересоздать. И заодно узнать много нового про билд процесс.

- Dotnet solution structure
- [[2024-01-19-MSBuild-operations|MSBuild operations]]
- [[2024-01-21-Shared-output-directory-for-dotnet-projects|Shared output directory]]

---
### Структура солюшена
Перед тем, как погрузиться в особенности работы билд процесса нужно понять что является входными аргументами этого процесса.

Если очень сильно обобщить и упроситить, то исходный код C# приложения имеет [[Dotnet project system|структуру]]:
```
SolutionDirectory/
	ProjectName/
		ProjectName.csproj
	Solution.sln
	Directory.Build.props
	Directory.Build.targets
```

Структура сформировалась вокруг необходимости группировать и разделять исходный код. С одной стороны, приложение - это много код, который удобно было бы держать в разных файлах. А процесс сборки соответственно требует сборки всех файлов вместе. Для реализации такого объединения существуют [[Dotnet project|проекты]].
С другой стороны, код нужно разделять на компоненты и блоки, а значит хочется иметь несколько проектов. Но всё ещё остаётся необходимость собирать все эти проекты вместе. Для объединения проектов существуют [[Dotnet solution|солюшены]], `.sln`  файлы.

Корень структуры - это [[Dotnet solution|солюшен]], `.sln` файл, который содержит ссылки на добавленные в него проекты (`.csproj` файлы). Они описываются так:

```
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") =
	"ProjectName",
	"ProjectName\ProjectName.csproj",
	"{E8F01267-DA18-42DC-9859-423209B99F3B}"
EndProject
```

где:
- "{9A19103F-16F7-4668-BE54-9A1E7A4F7556}" - это [[Dotnet project#Project type ID|тип проекта]]
- "ProjectName" - название проекта, как он отображается при открытии IDE
- "ProjectName\ProjectName.csproj" - путь к файлу .csproj относительно .sln
- "{E8F01267-DA18-42DC-9859-423209B99F3B}" - идентификатор проекта, сгенерированный GUID

Солюшены поддерживают возможность создавать [[Dotnet solution#Directory|директории]], которые используются для структурирования проектов в иерархию. Структура директорий в солюшене не привязывается к расположении проектов на файловой системе. 

Например, солюшен состоит из 8 проектов, из которых 3 - это тесты. Для них можно создать директорию Tests и иметь такую структуру:
```
Solutions
  Project1
  Project2
  ...
  Tests/
    TestProject6
    TestProject7
    TestProject8
```

Информация о созданных директориях хранится в sln файл. Для каждой директории создаётся запись аналогичная проекту:
```
Project("{2150E333-8FDC-42A3-9474-1A3956D46DE8}") =
	"Tests",
	"Tests",
	"{14195214-591A-45B7-851A-19D3BA2413F9}"
EndProject
```
А для каждого проекта, который создаётся в директории, создаётся запись в NestedProjects:
```
GlobalSection(NestedProjects) = preSolution
	{0031728E-A5D4-47C1-9C1A-6C859A765C9D} = {14195214-591A-45B7-851A-19D3BA2413F9}
```

### Конфигурация солюшена
Ещё одна секция в sln файле - это описание [[MSBuild build configuration and platform|конфигураций и платформ]] солюшена:
```
GlobalSection(SolutionConfigurationPlatforms) = preSolution
	Debug|Any CPU = Debug|Any CPU
	Release|Any CPU = Release|Any CPU
EndGlobalSection
```
Подробно о конфигурациях описано в документации Microsoft - https://learn.microsoft.com/en-us/visualstudio/ide/understanding-build-configurations. Выбор конфигурации солюшена влияет на то, какая конфигурация будет выбрана для проектов. Эта информация указывается в sln файле:
```
GlobalSection(ProjectConfigurationPlatforms) = postSolution
	{AF1E6F8A-5C63-465F-96F4-5E5F183A33B9}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
	{AF1E6F8A-5C63-465F-96F4-5E5F183A33B9}.Debug|Any CPU.Build.0 = Debug|Any CPU
```

Читать эту конфигурация нужно как:
> Для проекта "{AF1E6F8A-5C63-465F-96F4-5E5F183A33B9}" при выбранной для солюшена конфигурации "Debug|Any CPU" нужно использовать конфигурацию и платформу "Debug|Any CPU"

Вторая строка указывает на необходимость собирать проект. Если её убрать, то при сборке солюшена в "Debug|Any CPU" данный проект не будет собираться совсем. Пример использования: убрать тестовые проекты из процесса сборки при выбранной Release конфигурации.

### Структура проекта
[[Dotnet project|`.csproj` файлы]] имеют более сложную судьбу. Существует [[Dotnet project#File format|два формата их описания]]. Первый формат был создан для .NET Framework. Этот формат можно опознать по объявлению Project ноды:

```
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
```

Этот формат считается устаревшим. И вместе с .NET Core был добавлен новый формат, который называют SDK-style:

```
<Project Sdk="Microsoft.NET.Sdk">
```

Новый формат позиционировался как замена, поэтому все они имел схожий набор возможностей. Но поведение у них отличалось. Например, в старом формате записи требуется явно указывать все файлы, которые нужно скомпилировать. А вместе с выходом SDK-style появилось и стало использоваться по умолчанию свойство [[Dotnet project#EnableDefaultItems|EnableDefaultItems]], которое автоматически добавляло все \*.cs файлы в проект и в процесс компиляции.

Одним из основных элементов `.csproj` файла являются [[Dotnet project#Properties|Properties]]. Properties - это пары ключ-значение. В csproj файлы свойства указываются внутри ноды PropertyGroup:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <PropertyName>value</PropertyName>
  </PropertyGroup>
</Project>
```


Пример такой пары - указание версии dotnet'а `<TargetFramework>netstandard2.0</TargetFramework>`. В качестве ключа могут выступать не только стандартные имена свойств, которые заданы Microsoft'ом, но и любые пользовательские данные, которые можно использовать в качестве "переменных для MSBuild'а".

При описании свойств можно ссылаться на другие свойства используя специальный синтаксис. Например, OutputPath позволяет указывать путь, куда будут складываться [[MSBuild artifact directories|результаты сборки]]. По умолчанию это `/bin/debug/net8/...`.  При этом использование debug определяется выбранной конфигурацией и для Release будет использоваться Release. Записать это в csproj можно так:
```xml
<OutputPath>bin/$(Configuration)/$(TargetFramework)/</OutputPath>
```

Property можно дополнять условиями. Например, есть GeneratePackageOnBuild, которое указывает на то, нужно ли генерировать [[Файл .nupkg|.nupkg файл]] при сборке проекта. Чтобы включить генерацию, нужно прописать `<GeneratePackageOnBuild>True</GeneratePackageOnBuild>`. Но может появится запрос на то, чтобы генерировать файл только при сборке в Release конфигурации. Добиться такого можно таким изменением:

```xml
<GeneratePackageOnBuild Condition="$(Configuration) == 'Release'">True</GeneratePackageOnBuild>
```

Ещё одним важным элементом являются Item'ы. Item'ы - это свойства, которые являются списком элементов, которые передаются билд системе для сборки. Обычно, это одно из:
- Список `.cs` файлов, которые нужно скомпилировать (`<Compile Include = "Program.cs"/>`)
- Список нюгет пакетов, которые подключены в проект (`<PackageReference Include="System.Text.Json" />`)
- Список ссылок между проектами (`<ProjectReference Include="..\OtherProject\OtherProject.csproj" />`)

Запись вида `<PackageReference Include=...>` можно интерпретировать как "Добавить в список нюгет пакетов ещё одно значение". Item'ы указываются под ItemGroup нодой:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="NugetPackageName" />
  </ItemGroup>
</Project>
```
