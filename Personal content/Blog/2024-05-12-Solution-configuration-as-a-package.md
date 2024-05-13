---
title: Solution configuration as a package
---
## Контекст
Одна из проблем в разработке - это дублирование. Когда код дублируется, это означает, что любые изменения в нем потребуют внесения изменений в двух местах. В процессе эволюции разработки сформировалось понимание, что код нужно стараться переиспользовать, а не дублировать. Закрепился успех этой идеи вместе с распространением пакетных менеджеров, которые упростили возможность получать и обновлять пакеты с кодом. 
Но не везде от дублирования избавляются. Когда речь заходит о пакетах в dotnet, то сразу на ум приходят NuGet пакеты с C# кодом, различные библиотеки, которые подключаются к проектам, чтобы в них использовать предоставляемый API. Но помимо C# кода в проекте существуют ещё .csproj файлы. Далее будет рассмотрено, как можно применить пакетные менеджеры для уменьшения дублирования в таких файлах.

Две основные проблемы, которые решаются данным подходом:
- Упрощение поддержки существующих solution'ов. Время идёт, требования меняются, новые версии dotnet'а выходят. Ходить и обновлять каждый пакет, следить, чтобы изменения попадали во все репозитории сложно.
- Добавление новых пакетов. Чем больше надстроек над проектами, тем сложнее создавать новые проекты. Обычно появляются чек-листы создания новых проектов, которые состоят из большого количества шагов (скопировать что-то с существующего репозитория).

Пример пакета, который в итоге получился, лежит в открытом доступе: [kysect/SolutionDefaults: Kysect.SolutionDefaults is NuGet packages for distributing solution configuration via package manager (github.com)](https://github.com/kysect/SolutionDefaults).
## Какие данные нужно переиспользовать
`.csproj` файл описывает единицу сборки - проект. Он содержит много информации о конкретном проекте, которую не нужно переиспользовать: список файлов, связи с другими проектами, список используемых пакетов. Но есть и те, которые хочется задать на уровне всего solution'а или даже на уровне группы схожих solution'а.

Первая категория таких настроек была упомянута в [[2023-12-23-Introduction-to-Roslyn-analyzers-using]] - настройки анализатора. Там же была описана идея с использованием пакетного менеджера для их распространения - предшественник данного текста:

```xml
<PropertyGroup Label="Analyzers">
  <Nullable>enable</Nullable>
  <EnableNETAnalyzers>true</EnableNETAnalyzers>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <NoWarn>$(NoWarn);CS1591</NoWarn>
  <NoWarn Condition="$(IsTestProject) == 'true'">$(NoWarn);CA1707</NoWarn>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

Вторая категория - это настройка сборки. К ней можно причислить различные MSBuild свойства, которые задают поведение сборки: LangVersion, ImplicitUsings, [[MSBuild artifact directories#Artifacts output|UseArtifactsOutput]]. И с каждым релизом новой версии dotnet появляется что-то ещё:
```xml
<PropertyGroup Label="Build">
  <UseArtifactsOutput>true</UseArtifactsOutput>
  <LangVersion>latest</LangVersion>
  <ImplicitUsings>enable</ImplicitUsings>
  <SuppressNETCoreSdkPreviewMessage>true</SuppressNETCoreSdkPreviewMessage>
  <NeutralLanguage>en-US</NeutralLanguage>
</PropertyGroup>
```

Третья категория - настройки связанные с публикацией NuGet'ов - [sourcelink](https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/sourcelink), [snupkg](https://learn.microsoft.com/en-us/nuget/create-packages/symbol-packages-snupkg), [determeministic build](https://github.com/clairernovotny/DeterministicBuilds):
```xml
<PropertyGroup Label="Publish">
  <RepositoryType>git</RepositoryType>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
  <DebugType>portable</DebugType>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  <Deterministic>true</Deterministic>
</PropertyGroup>
```

Четвёртая категория - метаданные, которые прямого отношения к коду не имеют, то могут являться важной частью сборки - ссылка на репозиторий с кодом, список авторов, лицензия, лого:
```xml
<PropertyGroup Label="Metadata">
  <Authors>Kysect</Authors>
  <Company>Kysect</Company>
  <Copyright>(c) Kysect 2024</Copyright>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <RepositoryType>git</RepositoryType>
  <RepositoryUrl>https://github.com/kysect/SolutionDefaults</RepositoryUrl>
  <PackageProjectUrl>https://github.com/kysect/SolutionDefaults</PackageProjectUrl>
  <PackageReadmeFile>Readme.md</PackageReadmeFile>
  <PackageIcon>Icon.png</PackageIcon>
</PropertyGroup>
```

## Как NuGet'ы встраиваются в процесс сборки
Для простоты рассказа дальше будет использовать вполне конкретный пример - NuGet пакет Kysect.SolutionDefaults который добавляется в проект Kysect.CommonLib.

Почему использовать NuGet:
- Привычный для разработчиков инструмент
- Интегрирован с экосистемой dotnet, нет необходимости писать дополнительные скрипты, внедрять их в процесс сборки
- Версионирование из коробки

А дальше сложная часть всего процесса. Нужно разобраться с процессом попадания NuGet-пакета в проект и найти точки расширения, где можно добавить свою логику.

Рассмотрим шаги, которые проходит NuGet:

1. Добавление пакета в проект. Само добавление не оказывает влияния на проект и исходный код до начала выполнения команд MSBuild.
2. [[MSBuild#Solution restore|Restore]]. Во время выполнения Restore'а пакет скачивается из NuGet repository (обычно nuget.org) в директорию с [[Nuget#NuGet cache directory|NuGet caches]]. Сам nupkg файл - это архив, в котором находятся другие файлы. Этот архив распаковывается после скачивания, файлы раскладываются в `.nuget/packges/{package-name}/{package-version}`.
3. Во время компиляции MSBuild добавляет ссылку на распакованные `.dll` файлы
4. После компиляции выполняется [[MSBuild target|MSBuild target]] для копирования всех зависимостей в [[MSBuild artifact directories#bin directory|bin/]] или publish/ директорию.

## Доставляем .props файл
Из такого описания может показаться, что пакеты могут повлиять только на компиляции и добиться доставки конфигураций в `.csproj` не получится. Но у пакетов есть особенности, которые в большинстве обычных пакетов не используются. Внутри NuGet-пакета можно создать build/ директорию. Эта директория специальным образом обрабатывается MSBuild'ом. MSBuild пытается найти в ней файлы, которые называются именам пакета с расширениями .props и .target. Если такие файлы будут найдены, то во время Restore'а в obj директории будет сгенерирован для проекта файл ProjectName.csproj.nuget.g.props в котором среди прочего будет такой блок:
```xml
<ImportGroup Condition=" '$(ExcludeRestorePackageImports)' != 'true' ">
  <Import Project="$(NuGetPackageRoot)\kysect.solutiondefaults\0.1.2\
			  build\Kysect.SolutionDefaults.props"
  Condition="Exists('$(NuGetPackageRoot)\kysect.solutiondefaults\0.1.2\
			  build\Kysect.SolutionDefaults.props')" />
</ImportGroup>
```

Это значит, что во время выполнения MSBuild команд будет подключён .props файл из нашего пакета. А значит вклинится в сборку таки можно. Важным ограничением является то, что название `.props` файла должно соответствовать шаблону `{PackageName}.props`. В противном случае будет сгенерирована ошибка:
```
error NU5129: Warning As Error: - At least one .props file was found in 'build/', but'build/OtherName.props' was not. 
```

Окей, создаём проект Kysect.SolutionDefaults.csproj, создаём файл Kysect.SolutionDefaults.props, прописываем там все необходимые настройки, собираем пакет, добавляем в проект:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <IsPackable>true</IsPackable>
  </PropertyGroup>
  
  <ItemGroup>
    <None
		Include="Kysect.SolutionDefaults.props"
		Pack="true"
		PackagePath="/build/Kysect.SolutionDefaults.props" />
  </ItemGroup>
</Project>
```

Получаем ошибку связанную с `UseArtifactsOutput`:

```
error NETSDK1199: The ArtifactsPath and UseArtifactsOutput properties c 
annot be set in a project file, due to MSBuild ordering constraints. They must be set in a Directory.Build.props file or from the command line.
```

Эта проблема вызвана тем, что `.props` файл подключается в каждый проект независимо. `Directory.Build.props` концептуально работает таким же образом, то для него команда dotnet прописала отдельную логику, чтобы `UseArtifactsOutput` таки можно было выставить. А вот добавление `UseArtifactsOutput` в обычный проект (напрямую через `.csproj` или через сторонние `.props` файлы) будет заканчиваться с ошибкой. Это ограничение делает невозможным выставлять свойство `UseArtifatcsOutput` изнутри пакета.

Все остальные параметры должны работать корректно и будут учитываться при сборке проекта, к которому подключили пакет.

## Доставляем иконку для пакета
NuGet пакеты могут содержать иконку. Это ещё одна настройка, которую хотелось бы не настраивать в каждом отдельном проекте, а вместо этого сделать одну общую иконку и доставлять её пакетным менеджером. Сложность такого трюка в том, что сам файл нужно доставить пакетом, .props файла для этого недостаточно. В такой ситуации нужно прибегать ко второй "специальной" директории - `/content`. В эту директорию можно добавлять файлы, которые нужно запаковать, чтобы потом иметь к ним доступ:
```xml
<ItemGroup>
  <None
	  Include="../Images/Default-icon.png"
	  Pack="true"
	  PackagePath="/content/Default-icon.png" />
</ItemGroup>
```

Теперь до этого файла нужно добраться из MSBuild'а. Тут нужно вспомнить, что пакет - это архив, который разворачивается после скачивания и хранится в кэше. Точкой входа будет .props файл, который уже был добавлен в процесс сборки. Если открыть директорию, куда кэшируется пакет, то можно найти такую структуру:
```
/build/Kysect.SolutionDefaults.props
/content/Default-icon.png
/lib/...
Kysect.SolutionDefaults.x.y.z.nupkg
```

К счастью, MSBuild предоставляет удобный механизм, чтобы получить полный путь к текущему .props файлу - `$(MSBuildThisFileDirectory)`. А значит путь к иконке можно получить написав `$(MSBuildThisFileDirectory)../content/Default-icon.png`. Последний шаг - это прописать в .props файле добавление этой иконки:
```xml
<ItemGroup Label="DefaultIcon">
  <None Include="$(MSBuildThisFileDirectory)../content/Default-icon.png">
    <Visible>false</Visible>
    <Pack>true</Pack>
    <PackagePath>/Icon.png</PackagePath>
  </None>
</ItemGroup>
<PropertyGroup Label="DefaultIcon">
  <PackageIcon>Icon.png</PackageIcon>
</PropertyGroup>
```

`<None Include="...">` добавит иконку в пакет во время создания. `PackagePath` - это путь, где внутри пакета будет файл хранится, он должен быть указан в `PackageIcon`, чтобы использовать в качестве иконки пакета.
Важно понимать, что данный код добавляет элемент в проект, а значит этот элемент будет также отображаться в IDE как если бы файл лежал в директории с проектом. Чтобы файл не отображался в IDE можно выставить атрибут Visible в false.

## Доставляем .editorconfig и всё что угодно
Но возможность подложить .props файл открывает намного больше возможностей для интеграции со сборкой. Рассмотрим более сложную задачу. Нужно доставлять .editorconfig файл. Первый шаг аналогичен добавлению иконки:
```xml
<PropertyGroup>
  <NoDefaultExcludes>true</NoDefaultExcludes>
</PropertyGroup>

<ItemGroup>
  <None
	  Include=".editorconfig"
	  Pack="true"
	  PackagePath="/content/.editorconfig" />
</ItemGroup>
```

Важным отличием является необходимость добавить `NoDefaultExcludes`. MSBuild исключается из пакета все файлы, которые начинаются с точки. `NoDefaultExcludes` позволяет отключить это поведение. Теперь этот файл нужно скопировать в директорию solution'а, чтобы он применился. Это легко сделать зная, что в .props файле можно прописывать собственные target'ы, который MSBuild будет выполнять:
```xml
  <ItemGroup>
    <EditorConfigFilePath Include="$(MSBuildThisFileDirectory)../content/.editorconfig" />
  </ItemGroup>
  
  <!-- *Undefined* is deafult value when dotnet try to build one project -->
  <Target
    Name="SolutionDefaultsCopyEditorConfig"
    BeforeTargets="BeforeBuild"
    Condition="$(SolutionDir) != '*Undefined*' and $(SolutionDir) != '' and $(SolutionDefaultsCopyEditorConfig) != 'false'">
    <Copy
      SourceFiles="@(EditorConfigFilePath)"
      DestinationFolder="$(SolutionDir)"
      SkipUnchangedFiles="true"
      UseHardlinksIfPossible="false" />
  </Target>
```

Данный target будет вызывать MSBuild таску Copy, которая будет копировать файл в директорию solution'а. К сожалению, это будет работать только если выполнять сборку solution'а. При запуске сборки отдельного проекта переменная `$(SolutionDir)` не инициализируется и получить путь solution'а невозможно. (варианты с эвристикой и поиском по маске не рассматриваются). Поэтому лучше добавить условие пропуска target'а, если сборка запущена для проекта. Таким образом можно собрать один раз solution'а, а дальше можно запускать отдельно проекты и проблем не будет.

## Bonus: DevelopmentDependency
Полученный пакет является вспомогательным для сборки. В отличие от большинства обычных пакетов он не нужен в итоговой сборке, не нужен для работы приложения. И для подобных пакетов у MSBuild'а есть специальное свойство - `DevelopmentDependency`. Если выставить его в true, то MSBuild при добавлении пакета в другой проект будет автоматически добавлять атрибуты к `PackageReference`:
```xml
<PackageReference Include="Kysect.SolutionDefaults">
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

Выставив эти атрибуты пакет не будет транзитивно тянуться в другие, не будет копировать в bin/ и publish/. Хорошей практикой является добавлять такой атрибут к подобным пакетам, пакетам с compile-time информацией, анализаторам, генераторам.