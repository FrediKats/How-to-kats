---
title: MSBuild operations
---

Большинство инструментов, которые Microsoft делают для dotnet можно назвать "It's just work". Но иногда стандартной конфигурации недостаточно и приходиться более подробно разбираться в проблеме. Этот пост как раз о том, как стандартного процесса сборки солюшена оказалось недостаточно и пришлось его пересоздать. И заодно узнать много нового про билд процесс.

- [[2024-01-18-Dotnet-solution-structure|Dotnet solution structure]]
- MSBuild operations
- [[2024-01-21-Shared-output-directory-for-dotnet-projects|Shared output directory]]

---
### MSBuild Structured Log
Отправной точкой изучения [[MSBuild]] являются логи сборки. MSBuild поддерживает не только обычное текстовое логирование, но и структурное логирование. Для включения структурного логирования нужно выполнять команды сборки с аргументом `-bl`. Например, чтобы собрать [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) и получить структурные логи, можно использовать такие команды:
```cmd
git clone --depth 1 https://github.com/dotnet/BenchmarkDotNet.git
cd BenchmarkDotNet
dotnet restore -bl:restore.binlog BenchmarkDotNet.sln
dotnet build -bl:build.binlog BenchmarkDotNet.sln
```

В результате будут сгенерированы `.binlog` файлы, которые можно открыть используя [MSBuildStructuredLog](https://github.com/KirillOsenkov/MSBuildStructuredLog) (`winget install KirillOsenkov.MSBuildStructuredLogViewer`).

### MSBuild tasks and targets
[[MSBuild]] (The Microsoft Build Engine) - это платформа для сборки dotnet приложений.
Запустить сборку солюшена можно используя Visual Studio. Вместе с Visual Studio идёт msbuild.exe, который и отвечает за билд. Visual Studio использует .NET Framework реализацию MSBuild для загрузки и сборки проектов. Вместе с .NET Core появился альтернативный способ сборки - .NET Core реализация MSBuild, которая представлена [[Dotnet CLI|CLI командами]]. Новая функциональность, которую добавляют в MSBuild появляется и в Framework и в Core реализации, но .Framework содержит ряд API, которое изнчально не было перенесено в Core. Но это специфичные вещи (такие как использование COM объектов) и в большинстве случаев солюшен можно собрать как из Visual Studio, так и CLI командой.

Команды dotnet restore и dotnet build представляют собой набор вызываемых [[MSBuild target|таргетов]] и [[MSBuild task|тасок]].

[[MSBuild task]] - это единица выполнения в контексте MSBuild, которая выполняется во время сборки. MSBuild содержит ряд встроенных тасок (например, `<MakeDir Directories="$(BuildDir)" />`  для создания директории), а также предоставляет возможность создавать собственные таски.

[[MSBuild target]] - это группы тасок, которые объединяются для реализации более сложных сценариев. Пример таргета, который вызывается во время `dotnet restore`:
```xml
<Target Name="ValidateSolutionConfiguration">
	<Error
		Condition="('$(CurrentSolutionConfigurationContents)' == '')
				and ('$(SkipInvalidConfigurations)' != 'true')"
		Text="The specified solution configuration &quot;$(Configuration)|$(Platform)&quot; is invalid." />
	<Warning
		Condition="('$(CurrentSolutionConfigurationContents)' == '')
			and ('$(SkipInvalidConfigurations)' == 'true')"
		Text="The specified solution configuration &quot;$(Configuration)|$(Platform)&quot; is invalid." />
	<Message
		Condition="'$(CurrentSolutionConfigurationContents)' != ''"
		Text="Building solution configuration &quot;$(Configuration)|$(Platform)&quot;." />
</Target>
```

Error - это таска, которая фиксирует сообщение об ошибке сборки. Такая ошибка ведёт к не успешному завершению процесса сборки, следующие шаги не будут выполняться. В контексте ValidateSolutionConfiguration Error используется вместе с условием `('$(CurrentSolutionConfigurationContents)' == '')`, а значит под валидацией подразумевается проверка на то, что `$(CurrentSolutionConfigurationContents)` должен быть задан. Вторая часть проверки - это проверка флага SkipInvalidConfigurations. Этот флаг имеет смысл, если рассматривать вместе со следующей таской Warning. Она выполняет такую же валидацию, но вместо ошибки создаётся предупреждение и сборка может выполняться далее. Грубо говоря, эту же логику можно было записать кодом так:
```csharp
if (CurrentSolutionConfigurationContents == "")
  if (!SkipInvalidConfigurations)
    throw new Exception("The specified solution configuration is invalid");
  else
    logger.Warning("The specified solution configuration is invalid");
else
  logger.Message("Building solution configuration $"{Configuration}|{Platform}.");
```

В бинлоге можно найти результат выполнения:
```
Target Name=ValidateSolutionConfiguration Project=BenchmarkDotNet.sln
  Task "Error" skipped, due to false condition...
  Task "Warning" skipped, due to false condition...
  Message: Building solution configuration "Debug|Any CPU".
```

Таргеты могут иметь возвращаемое значение, для этого в описании таргета используется атрибут `Returns`. Пример таргета с возвращаемым значением - \_LoadRestoreGraphEntryPoints. Его задача - сформировать список проектов, которые нужно ресторить. Рассматривается два сценария:
- Рестор одного проекта `dotnet restore Project.csproj`. В таком случае в список проектов для рестора добавляется текущий проект: `<RestoreGraphProjectInputItems Include="$(MSBuildProjectFullPath)" />`.
- Рестор солюшена `dotnet restore Solution.sln`. В таком случае нужно из sln получить список проектов. Для этого выполняется таска GetRestoreSolutionProjectsTask и её output сохраняется в переменную, которая является `Returns` таргета.

```xml
<!--
	============================================================
	_LoadRestoreGraphEntryPoints
	Find project entry points and load them into items.
	============================================================
-->
<Target Name="_LoadRestoreGraphEntryPoints" Returns="@(RestoreGraphProjectInputItems)">

	<!-- Project case -->
	<ItemGroup
	Condition=" $(MSBuildProjectFullPath.EndsWith('.metaproj')) != 'true'
			AND @(RestoreGraphProjectInputItems) == '' ">
		  <RestoreGraphProjectInputItems Include="$(MSBuildProjectFullPath)" />
	</ItemGroup>

	<!-- Solution case -->
	<GetRestoreSolutionProjectsTask
		Condition=" $(MSBuildProjectFullPath.EndsWith('.metaproj')) == 'true'
			  AND @(RestoreGraphProjectInputItems) == '' "
		ProjectReferences="@(ProjectReference)"
		SolutionFilePath="$(MSBuildProjectFullPath)">
	<Output
		TaskParameter="OutputProjectReferences"
		ItemName="RestoreGraphProjectInputItems" />
	</GetRestoreSolutionProjectsTask>
</Target>
```

Бинлог сохраняет информацию о результатах выполнения таски и позволяет посмотреть значение, которое было возвращено через Returns:
```
Add item RestoreGraphProjectInputItems
  ..\src\BenchmarkDotNet\BenchmarkDotNet.csproj
  ..\src\BenchmarkDotNet.Diagnostics.Windows\BenchmarkDotNet.Diagnostics.Windows.csproj
  ..\samples\BenchmarkDotNet.Samples.FSharp\BenchmarkDotNet.Samples.FSharp.fsproj
  ...
```

Вызвать таргет и получить возвращаемое значение можно используя команду CallTarget:
```xml
<CallTarget Targets="_LoadRestoreGraphEntryPoints">
	<Output
		TaskParameter="RestoreGraphProjectInputItems"
		ItemName="VariableForReturnValues" />
</CallTarget>
```

В данном примере `RestoreGraphProjectInputItems` - это название возвращаемого значения, которое прописано в Returns, а `VariableForReturnValues` - это переменная, куда значения нужно сохранить.
### dotnet restore: project.assets.json
Основным таргетом команды рестора является RestoreTask. Задача RestoreTask - сформировать список используемых нюгет пакетов в проектах и загрузить их. Есть два основных артефакта работы этой таски:
- [[MSBuild artifact directories#project.assets.json]]] файлы для каждого проекта
- Скачанные локально нюгеты в директории C:\\Users\\fredi\\.nuget\\packages\\

Рассмотрим рестор на примере простого солюшена из 4 проектов:
- ProjectA
- ProjectB
- ProjectC, который зависит от ProjectA и ProjectB
- ProjectD, который зависит от ProjectC

Если выполнить `dotnet restore` и открыть `ProjectA/obj/project.assets.json`, то можно обнаружить шаблонный почти пустой файл, который содержит метаинформацию о проекте.

Добавим в ProjectA нюгет Microsoft.Extensions.Logging.Abstractions версии 7.0.0 и выполним ещё один restore. После этого в проектах A, C и D появятся упоминания этого нюгета:
```json
"libraries": {
	"Microsoft.Extensions.Logging.Abstractions/7.0.0": {
```

Следующий шаг - добавление Microsoft.Extensions.Logging.Abstractions версии 8.0.0 в ProjectB. Наблюдаем изменения в проектах ProjectB, ProjectC, ProjectD, теперь там указывается версия 8.0.0:
```json
  "libraries": {
    "Microsoft.Extensions.DependencyInjection.Abstractions/8.0.0": {
```

Это работает за счёт того, что nuget'ы по умолчанию ресторятся к старшей версии без конфликтов. Но если в ProjectD добавить зависимость на версию 7.0.0, то рестор закончится с ошибкой:

```
error NU1605: Warning As Error: Detected package downgrade: Microsoft.Extensions.Logging from 8.0.0 to 7.0.0. Reference the package directly from the project to select a different version.]
error NU1605:  ProjectD -> ProjectC -> Microsoft.Extensions.Logging (>= 8.0.0)
error NU1605:  ProjectD -> Microsoft.Extensions.Logging (>= 7.0.0)
```

Более подробно про логику резолва версий можно узнать с документации - https://learn.microsoft.com/en-us/nuget/concepts/dependency-resolution.

### dotnet restore: framework dependencies
Вторая задача рестора - скачать необходимые нюгеты в локальный кеш. Кеш по умолчанию находится в директории `C:\Users\User\.nuget\packages\`. Внутри пакет может храниться сразу в нескольких версиях. Например:

```
microsoft.extensions.logging/
	2.1.1/*
	7.0.0/*
	8.0.0/
		lib/
			net462/
			netstandard2.0/
			net8.0/
		
```

Нюгеты собираются под разные фреймворки для того, чтобы их можно было подключить к максимально большому количеству проектов. С точки зрения нюгетов основное отличие фреймворков - это набор доступного API. На сайте https://dotnet.microsoft.com/en-us/platform/dotnet-standard можно сравнить количество доступных методов в standard 2.0 и standard 2.1.

Но фреймворк - это не всегда приговор. Некоторое API backport'ят в старые версии с помощью нюгетов. Это в результате и создаёт разный набор зависимых нюгетов. Например, в Microsoft.Extensions.Logging для .NET Framework 4.6.2 есть такие зависимости:
- Microsoft.Bcl.AsyncInterfaces
- System.ValueTuple
- System.Diagnostics.DiagnosticSource

Но этих же зависимостей нет в .NET 8 потому что эти нюгеты являются частью стандартной конфигурации .NET 8.

Во время рестора фреймворк выбирается исходя из версии проекта, куда нужно нюгет подключить. Если проект версии 8.0.0, то среди доступных версий будет искаться версия для .NET 8, .NET 7 и так далее. Если на будет найдена версия для .NET Core, то будет искаться версия для .NET standard. Если не будет найдена и такая версия, то будет взята версия .NET Framework. Но такой рестор является не безопасным т.к. не всё API из Framework доступно в .NET 8 и рестор будет заканчиваться с предупреждениями. Проверить какая версия была выбрана можно в `project.assets.json`, там будет указан относительный путь к dll: `lib/net8.0/Microsoft.Extensions.Logging.dll`

Более подробно описано тут: https://learn.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks.

### Формирование списка зависимостей
Команда dotnet build состоит из большего количества таргетов и тасок. Дополнительная сложность в анализе состоит в том, что основной таргет `Build` вызывает таску MSBuild, которая в свою очередь запускает параллельный процесс сборки проектов:
```xml
<Target Name="Build" Outputs="@(CollectedBuildOutput)">
  <MSBuild
	  BuildInParallel="True"
	  Properties="...."
	  Projects="@(ProjectReference)">
      <Output
	      TaskParameter="TargetOutputs"
	      ItemName="CollectedBuildOutput" />
    </MSBuild>
  </Target>
```

Далее будет рассмотрен билд одного конкретного проекта - BenchmarkDotNet.IntegrationTests. Пропустим несколько не значимых тарегтов и начнём с ResolvePackageAssets. Основная задача этого таргета - сформировать список зависимостей, файлов, которые проект ожидает получить из нюгет пакетов. Для этого вызывается таска [[MSBuild tasks ResolvePackageAssets|ResolvePackageAssets]]:

```xml
<ResolvePackageAssets
  ProjectAssetsFile="$(ProjectAssetsFile)"
  ProjectAssetsCacheFile="$(ProjectAssetsCacheFile)"
  ProjectPath="$(MSBuildProjectFullPath)"
  ProjectLanguage="$(Language)"
  CompilerApiVersion="$(CompilerApiVersion)"
  EmitAssetsLogMessages="$(EmitAssetsLogMessages)"
  TargetFramework="$(TargetFramework)"
  RuntimeIdentifier="$(RuntimeIdentifier)"
  PlatformLibraryName="$(MicrosoftNETPlatformLibrary)"
  RuntimeFrameworks="@(RuntimeFramework)"
  IsSelfContained="$(SelfContained)"
  MarkPackageReferencesAsExternallyResolved="$(MarkPackageReferencesAsExternallyResolved)"
  DisablePackageAssetsCache="$(DisablePackageAssetsCache)"
  DisableFrameworkAssemblies="$(DisableLockFileFrameworks)"
  CopyLocalRuntimeTargetAssets="$(CopyLocalRuntimeTargetAssets)"
  DisableTransitiveProjectReferences="$(DisableTransitiveProjectReferences)"
  DisableTransitiveFrameworkReferences="$(DisableTransitiveFrameworkReferences)"
  DotNetAppHostExecutableNameWithoutExtension="$(_DotNetAppHostExecutableNameWithoutExtension)"
  ShimRuntimeIdentifiers="@(_PackAsToolShimRuntimeIdentifiers)"
  EnsureRuntimePackageDependencies="$(EnsureRuntimePackageDependencies)"
  VerifyMatchingImplicitPackageVersion="$(VerifyMatchingImplicitPackageVersion)"
  ExpectedPlatformPackages="@(ExpectedPlatformPackages)"
  SatelliteResourceLanguages="$(SatelliteResourceLanguages)"
  DesignTimeBuild="$(DesignTimeBuild)"
  ContinueOnError="$(ContinueOnError)"
  PackageReferences="@(PackageReference)"
  DefaultImplicitPackages= "$(DefaultImplicitPackages)">

  <!-- NOTE: items names here are inconsistent because they match prior implementation
	  (that was spread across different tasks/targets) for backwards compatibility.  -->
  <Output TaskParameter="Analyzers" ItemName="ResolvedAnalyzers" />
  <Output TaskParameter="ApphostsForShimRuntimeIdentifiers" ItemName="_ApphostsForShimRuntimeIdentifiersResolvePackageAssets" />
  <Output TaskParameter="ContentFilesToPreprocess" ItemName="_ContentFilesToPreprocess" />
  <Output TaskParameter="DebugSymbolsFiles" ItemName="_DebugSymbolsFiles" />
  <Output TaskParameter="ReferenceDocumentationFiles" ItemName="_ReferenceDocumentationFiles" />
  <Output TaskParameter="FrameworkAssemblies" ItemName="ResolvedFrameworkAssemblies" />
  <Output TaskParameter="FrameworkReferences" ItemName="TransitiveFrameworkReference" />
  <Output TaskParameter="NativeLibraries" ItemName="NativeCopyLocalItems" />
  <Output TaskParameter="ResourceAssemblies" ItemName="ResourceCopyLocalItems" />
  <Output TaskParameter="RuntimeAssemblies" ItemName="RuntimeCopyLocalItems" />
  <Output TaskParameter="RuntimeTargets" ItemName="RuntimeTargetsCopyLocalItems" />
  <Output TaskParameter="CompileTimeAssemblies" ItemName="ResolvedCompileFileDefinitions" />
  <Output TaskParameter="TransitiveProjectReferences" ItemName="_TransitiveProjectReferences" />
  <Output TaskParameter="PackageFolders" ItemName="AssetsFilePackageFolder" />
  <Output TaskParameter="PackageDependencies" ItemName="PackageDependencies" />
  <Output TaskParameter="PackageDependenciesDesignTime" ItemName="_PackageDependenciesDesignTime" />
</ResolvePackageAssets>
```

Вызывая эту таску можно получить:
- Analyzers, например:
	- Microsoft.CodeAnalysis.Analyzers.dll
	- xunit.analyzers.dll
- DebugSymbolsFiles - пути к сгенерированным pdb символам:
	- Microsoft.Diagnostics.Runtime.pdb
	- Mono.Cecil.pdb
- RuntimeAssemblies -  пути к dll, которые получены из нюгет пакетов:
	- CommandLine.dll
	- Microsoft.Extensions.Options.dll
- RuntimeTargets - platform specific files:
	- gee.external.capstone\\2.3.0\\runtimes\\linux-arm\\native\\libcapstone.so
	- gee.external.capstone\\2.3.0\\runtimes\\linux-arm64\\native\\libcapstone.so

Помимо зависимостей от нюгетов, есть также зависимость от других проектов. Таргет GetCopyToOutputDirectoryItems позволяет получить список файлов других проектов, от которых зависит текущий: 
```xml
<Target
  Name="GetCopyToOutputDirectoryItems"
  Returns="@(AllItemsFullPathWithTargetPath)"
  KeepDuplicateOutputs=" '$(MSBuildDisableGetCopyToOutputDirectoryItemsOptimization)' == '' "
  DependsOnTargets="$(GetCopyToOutputDirectoryItemsDependsOn)">
	<CallTarget
		Targets="_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences">
	  <Output
		  TaskParameter="TargetOutputs"
		  ItemName="_TransitiveItemsToCopyToOutputDirectory" />
	</CallTarget>

	<CallTarget
		Targets="_GetCopyToOutputDirectoryItemsFromThisProject">
	  <Output
		  TaskParameter="TargetOutputs"
		  ItemName="_ThisProjectItemsToCopyToOutputDirectory" />
	</CallTarget>

</Target>
```

Таргет состоит из вызова двух других таргетов: поиск файлов для текущего проекта и поиск файлов для всех зависимостей (в том числе транзитивно). Пример файлов, которые возвращает `GetCopyToOutputDirectoryItems`:
- .deps.json и .runtime.config.json файлы
- .dll и .exe файлы с результатами компиляции проекта
- Добавленные в проект файлы, которые требуют Copy to output directory.
 
### [[MSBuild artifact directories#obj directory|obj]] и [[MSBuild artifact directories#bin directory|bin]] директории
В процессе компиляции из исходного кода проекта создаётся dll. Происходит это в рамках таргета CoreCompile, где вызывается таска Csc. Одним из аргументов этой таски пеердаётся OutputAssembly - это путь, куда нужно сохранить скомпилированную dll. По умолчанию компиляция работает с obj директориями. При этом в obj директорию попадают только те файлы, которые были сгенерированы в процессе компиляции. Нет необходимости копировать туда зависимости, они указываются в csc как пути к .nuget/ или obj/ других проектов.

Запустить dll из директории obj невозможно. Чтобы собранное приложение можно было запустить выполняется копирование этой длл и всех зависимостей в директорию bin. Для этого выполняется таргет CopyFilesToOutputDirectory.

Таргет CopyFilesToOutputDirectory декомпозируется на несколько шагов. Один из шагов - это копирование файлов-зависимостей таргетом \_CopyFilesMarkedCopyLocal. Он внутри себя вызывает таску Copy для того, чтобы в директорию `bin\` скопировать содержимое нюгет пакетов, pdb-файлы и подобное:
```
Target Name=_CopyFilesMarkedCopyLocal Project=BenchmarkDotNet.IntegrationTests.csproj
  Task Copy
    Parameters
      SourceFiles
        C:\Users\User\.nuget\packages\argon\0.7.2\lib\net8.0\Argon.dll
      DestinationFiles
        bin\Debug\net8.0\Argon.dll
    Copying file from "...\lib\net8.0\Argon.dll" to "...\bin\Debug\net8.0\BenchmarkDotNet.IntegrationTests.DisabledOptimizations.dll".
```

После выполнения копирования можно считать, что основная работа выполнена и проект готов к тому, чтобы запускаться из bin директории.
