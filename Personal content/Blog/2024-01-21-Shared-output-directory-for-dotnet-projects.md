---
title: Shared output directory for dotnet projects
---

Большинство инструментов, которые Microsoft делают для dotnet можно назвать "It's just work". Но иногда стандартной конфигурации недостаточно и приходиться более подробно разбираться в проблеме. Этот пост как раз о том, как стандартного процесса сборки солюшена оказалось недостаточно и пришлось его пересоздать. И заодно узнать много нового про билд процесс.

- [[2024-01-18-Dotnet-solution-structure|Dotnet solution structure]]
- [[2024-01-19-MSBuild-operations|MSBuild operations]]
- Shared output directory

---
## Одна bin директория для нескольких проектов
Рост размера солюшена сказывается на билд процесс. С увеличением количества проектов не только увеличивается время сборки, но и размер создаваемых [[MSBuild artifact directories#bin directory|bin директорий]]. Представим ситуацию, когда есть солюшен с 300 проектами. Эти проекты могут иметь много зависимостей как внутри себя, так и от бинарных файлов вне солюшена. Если размер bin директории будет в среднем 300 мегабайт, то после компиляции получаем -90 гигабайт свободного места.

Добавим несколько деталей про солюшен. Предположим, что солюшен состоит из нескольких запускаемых сервисов, а остальные проекты представляют собой общий код, используемый этими сервисами. Отсюда можно сделать предположение - достаточно формировать bin директории только для сервисов. Остальные же проекты можно объединить в одну директорию для экономии места.

Внезапно оказывается, что сценарий с общей bin директорией не полностью поддерживается MSBuild'ом. На GitHub'е можно найти созданные ишуи на эту тему, например - https://github.com/dotnet/sdk/issues/17645. Проблема заключается в том, что MSBuild пытается обрабатывать проекты параллельно. Представим ситуацию, что есть:
- Проект1
- Проект2, который зависит от Проект1
- Проект3, который зависит от Проект1

В такой ситуации Проект1 будет собран первым, и пока он не будет собран, остальные не могу начать сборку. Но после этого появляется два кандидата на сборку, которые независимо могут собираться. Но если добавить для этих двух проектов общую bin директорию, то получается, что Проект2 и Проект3 будут параллельно пытаться скопировать свои зависимости в одну директорию, включая файлы от Проект1. Но ситуация ещё больше усложняется, если посмотреть на [[MSBuild structured log]] и найти там шаги очистки временных файлов во время сборки. Например, билд процесс создаёт файлы .deps.json и .runtimeconfig.json, который удаляются в конце. Но если проекты собираются параллельно, то один проект может удалить файлы, а второй ещё не успеет закончить работать с ними. Вторая возможная проблема - попытка двух проектов одновременно начать создавать один и тот же файл, что приводит к ошибке `Access to the path is denied.`

## Common Output Directory
Microsoft всё же предоставили решение для этой проблемы - Common Output Directory. Это [[Dotnet project#Properties|свойство проекта]], которое указывает [[MSBuild]]'у на то, что будет использована общая bin директория для многих проектов. MSBuild в свою очередь пытается предотвратить конфликты с копированием файлов: он их не копирует, копируются только артефакты сборки конкретного проекта.

Код таргета \_CopyFilesMarkedCopyLocal:
```xml
<Target
  Name="_CopyFilesMarkedCopyLocal"
  Condition="'@(ReferenceCopyLocalPaths)' != ''">
<!-- Skip some details -->
  <Copy
    SourceFiles="@(ReferenceCopyLocalPaths)"
    ...
    Condition="'$(UseCommonOutputDirectory)' != 'true'">
  </Copy>
</Target>
```

Код таргета \_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences:
```xml
<Target
  Name="_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences"
  ...
  >

<!-- Get items from child projects first. -->
<MSBuild
	...
	Condition="
		'@(_MSBuildProjectReferenceExistent)' != ''
		and '$(_GetChildProjectCopyToOutputDirectoryItems)' == 'true'
		and '%(_MSBuildProjectReferenceExistent.Private)' != 'false'
		and '$(UseCommonOutputDirectory)' != 'true'"
	...
	>
</MSBuild>
```

Но как работает приложение, если зависимости не копируются? Для начала стоит разобраться с зависимостями между проектами. Идея Common output в том, что будет копироваться всё в одну директорию. А значит, если все проекты скопируют свои DLL туда, то все необходимые DLL будут в одной директории.. Проблема решена. Единственный нюанс в том, что использование Common Output не влияет на Output path проектов и если у проектов будут разные пути, то магии не случится.

С нюгетами история ещё более грустная - они просто не копируются в Output. Как итог, приложение из Output не может быть запущено. Соответственно, тесты не будут работать, на это есть ишуя - https://github.com/dotnet/sdk/issues/27320. На данный момент из предложенных решений только выключение Common Output для тестов - `Condition="!$(MSBuildProjectName.EndsWith('Test'))"`.

Но проблема аффектит не только тесты, а все проекты. А значит для решения нужно... Отключить Common Output везде! Но есть менее радикальное решение - использовать dotnet publish. Процесс паблишинга не пропускает нюгета, а копирует все нужные файлы и после такого копирования. Процесс публикации параллельно будет пытаться скопировать файлы в нужную директорию, а значит, возвращает нас к проблеме с потенциальными одновременными записями нескольких DLL в одну директорию. Эта проблема описана в issue https://github.com/dotnet/core/issues/2524.

## Берём ситуация в свои руки - пишем таргет
У Microsoft есть много причин, по которым они не хотят поддерживать сценарии с копированием в одну директорию. Например, MSBuild умеет находить double writes - файл, который во время билда попытались создать дважды. Например, если два проект использую NuGet пакет разных версий и складываются в одну директорию, то они будут пытаться записать DLL разных версий в output. Это может приводить к тому, что один из двух проектов не будет работать. Но если не обращать внимание на такие проблемы, поверить, что [[Dotnet Central Package Management|Central Package management]] справится, то можно сделать собственную реализацию алгоритма копирования файлов в Output.

Определим задачу так:
Нужно реализовать алгоритм сборки таким образом, чтобы солюшен из 300+ проектов мог собираться в несколько пересекающихся директорий. В результате сборки должны получаться директории, в которых лежат dll со всеми зависимостями.

С точки зрения реализации можно переформулировать:
В логику сборки проекта нужно добавить механизм, который бы находил все зависимости и копировал его в output проекта.

Первый шаг - создать тестовый солюшен:
```powershell
mkdir SampleSolution
cd SampleSolution
dotnet new buildprops --use-artifacts
dotnet new sln --name SampleSolution
dotnet new console --output Project1
dotnet new console --output Project2

dotnet sln add Project1
dotnet sln add Project2
dotnet add Project1 reference Project2
dotnet add Project1 package Microsoft.Extensions.DependencyInjection
dotnet add Project2 package Microsoft.Extensions.Logging
```

Второй шаг - включение в солюшене Common Output. Несмотря не описанные выше проблемы, Common output является полезным из-за того, что он отключает стандартное копирование, а значит решает проблему с гонками. Всё что остаётся - сделать за него его работу, но с учётом возможных гонок.

Зафиксируем список файлов, которые попадают в Output до включения Common Output:
```powershell
dotnet build
ls artifacts\bin\Project1\Debug | select-object -expandproperty Name
```

Должны получить такой список:
```
Microsoft.Extensions.DependencyInjection.Abstractions.dll
Microsoft.Extensions.DependencyInjection.dll
Microsoft.Extensions.Logging.Abstractions.dll
Microsoft.Extensions.Logging.dll
Microsoft.Extensions.Options.dll
Microsoft.Extensions.Primitives.dll
Project1.deps.json
Project1.dll
Project1.exe
Project1.pdb
Project1.runtimeconfig.json
Project2.deps.json
Project2.dll
Project2.exe
Project2.pdb
Project2.runtimeconfig.json
```

Проверяем, что Common Output действительно выключает копирование:
```
 rm artifacts -r -force
 // Add <UseCommonOutputDirectory>true</UseCommonOutputDirectory>
 // Idk CLI command for this
 // Time to create it!
 dotnet build
 ls artifacts\bin\Project1\Debug | select-object -expandproperty Name
```

После включения UseCommonOutputDirectory в директории остались только файлы Project1.\*.

Шаг 3. Научиться доставать список нюгетов для копирования. Если вернуться к binlog'у и вниматель посмотреть, то можно обнаружить таргет GetCopyToOutputDirectoryItems. Они делает за нас всю работу, нужно лишь получить из него данные. Для этого можно написать свой таргет, который будет вызываться после `GetCopyToOutputDirectoryItems` и сохранять результаты его выполнения. В корне солюшена нужно создать Directory.Build.targets:
```xml
<Project>
  <Target
    Name="SaveResults"
    AfterTargets="GetCopyToOutputDirectoryItems">
    <ItemGroup>
      <FilesForCopy Include="@(ReferenceCopyLocalPaths)" />
    </ItemGroup>
    <Message
      Text="$(ProjectName) saved dependencies: @(ReferenceCopyLocalPaths)"
      Importance="High" />
  </Target>
</Project>
```

Данный таргет будет автоматически вызываться для каждого проекта и используя таску Message писать в консоль список всех зависимостей. Также, убедиться, что таргет вызывается и в более читаемом виде увидеть результаты можно в бинлоге:
```
dotnet build --no-incremental -bl
```

Результат должен быть таким:
```
Project1 saved dependencies:
-...\Microsoft.Extensions.DependencyInjection.dll
-...\Microsoft.Extensions.DependencyInjection.Abstractions.dll
-...\Microsoft.Extensions.Logging.dll
-...\Microsoft.Extensions.Logging.Abstractions.dll
-...\Microsoft.Extensions.Options.dll
-...\Microsoft.Extensions.Primitives.dll
-SampleSolution\artifacts\bin\Project2\debug\Project2.dll
-SampleSolution\artifacts\bin\Project2\debug\Project2.pdb
```

Внимательные читатели могли заметить, что в списке не хватает как минимум одного файла - Project2.exe. А самые внимательные могли ещё и сопоставить этот момент с указанным ранее таргетом \_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences. Этот таргет отвечает за то, чтобы сформировать список файлов, который нужно скопировать из проектов-зависимостей. И внутри этого таргета перед вызов MSBuild таски есть условие на UseCommonOutputDirectory, которое приводит к скипу выполнения в нашем сценарии. Но мы уже решили, что берём ситуацию в свои руки, поэтому скопируем вызов MSBuild из этого таргета в наш таргет и уберём "лишнее условие":
```xml
<MSBuild
  Projects="@(_MSBuildProjectReferenceExistent)"
  Targets="$(_RecursiveTargetForContentCopying)"
  BuildInParallel="$(BuildInParallel)"
  Properties="
	  %(_MSBuildProjectReferenceExistent.SetConfiguration);
	  %(_MSBuildProjectReferenceExistent.SetPlatform);
	  %(_MSBuildProjectReferenceExistent.SetTargetFramework)"
  Condition="
	  '@(_MSBuildProjectReferenceExistent)' != ''
	  and '$(_GetChildProjectCopyToOutputDirectoryItems)' == 'true'
	  and '%(_MSBuildProjectReferenceExistent.Private)' != 'false'"
  ContinueOnError="$(ContinueOnError)"
  SkipNonexistentTargets="true"
  RemoveProperties="%(_MSBuildProjectReferenceExistent.GlobalPropertiesToRemove)$(_GlobalPropertiesToRemoveFromProjectReferences)">
  <Output TaskParameter="TargetOutputs" ItemName="TransitiveProjectsFiles"/>
</MSBuild>
```

Перезапустим билд и увидим, что при вызове нашего таргета из потока процесса сборки Project1 в TransitiveProjectsFiles попали 3 файла:
- SampleSolution\\artifacts\\bin\\Project2\\debug\\Project2.deps.json
- SampleSolution\\artifacts\\bin\\Project2\\debug\\Project2.runtimeconfig.json
- SampleSolution\\artifacts\\obj\\Project2\\debug\\apphost.exe

Прогресс есть, но последний вызывает вопросы. Что ещё за apphost.exe? Apphost.exe - это стандартное название для .exe файлов, которые создаются во время компиляции. И переименовываются они в "правильные" названия в процессе копирования в bin директорию. Если открыть binlog и посмотреть на эти файлы, то можно заметить, что это не просто строковые значения с путями, они содержат ряд свойств, например:
- CopyToOutputDirectory = PreserveNewest
- TargetPath = Project2.exe

Можно считать, что вся нужная информация есть, нужно будет только разобраться, как её использовать.

## Берём ситуация в свои руки - пишем таску
MSBuild предоставляет API для расширения стандартного процесса. Помимо добавления собственных таргетов, MSBuild даёт возможность написать свои таски и включить их в билд процесс. 

Создаём солюшен:
```powershell
mkdir SampleSolutionBuildTask
cd SampleSolutionBuildTask
dotnet new buildprops --use-artifacts
dotnet new sln --name SampleSolutionBuildTask
dotnet new classlib --output SampleSolutionBuildTask --framework "netstandard2.0"
dotnet sln add SampleSolutionBuildTask
dotnet add SampleSolutionBuildTask package Microsoft.Build.Framework
dotnet add SampleSolutionBuildTask package Microsoft.Build.Utilities.Core
```

Microsoft.Build.Framework - это Nuget пакет, который предоставляет нужно API. Точкой входа в написание таски является класс Task от которого можно отнаследовать свою реализацию:
```cs
public class CopyLocalWithMutex : Microsoft.Build.Utilities.Task
{
  // ...
}
```

Для того, чтобы выполнить копирование, в таску нужно передать 3 аргумента:
- OutputPath проекта
- Список item'ов из ReferenceCopyLocalPaths
- Список item'ов из \_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences

ITaskItem - это интерфейс из библиотеки MSBuild, который описывает экземпляр передаваемых аргументов или возвращаемых значений между тасками. Для добавления аргумента в таску достаточно создать публичное поле, его можно также пометить как Required, чтобы MSBuild сам требовал явного указания аргумента при вызове:
```csharp
[Required]
public string OutputDirectoryPath { get; set; }

[Required]
public ITaskItem[] ReferenceCopyLocalPaths { get; set; }

[Required]
public ITaskItem[] CopyToOutputFromDependencies { get; set; }
```

Следующий шаг - реализация копирования с [[Mutex|мьютексами]], ради которой всё и затевалось:
```csharp
private void Copy(string source, string target)
{
    String mutexName = "FS_Mutex_" + target.Replace(Path.DirectorySeparatorChar, '_');
    var mutex = new Mutex(false, mutexName);
    mutex.WaitOne();

    if (File.Exists(target))
        File.Delete(target);
    File.Copy(source, target);

    mutex.ReleaseMutex();
    mutex.Dispose();
}
```

И вызов этого копирования:
```csharp
public override bool Execute()
{
    var copyTasks = new List<(string Source, string Target)>();

    foreach (var referenceCopyLocalPath in ReferenceCopyLocalPaths)
    {
        var sourcePath = referenceCopyLocalPath.ToString();
        var fileInfo = new FileInfo(sourcePath);
        var targetPath = Path.Combine(OutputDirectoryPath, fileInfo.Name);

        copyTasks.Add((sourcePath, targetPath));
    }

    foreach (var copyToOutputFromDependency in CopyToOutputFromDependencies)
    {
        var sourcePath = copyToOutputFromDependency.ToString();
        var partialTargetPath = copyToOutputFromDependency.GetMetadata("TargetPath");
        var targetPath = Path.Combine(OutputDirectoryPath, partialTargetPath);

        copyTasks.Add((sourcePath, targetPath));
    }

    foreach (var copyTask in copyTasks)
    {
        Log.LogMessage($"Copy {copyTask.Source} to {copyTask.Target}");
        Copy(copyTask.Source, copyTask.Target);
    }

    return true;
}
```

И этого достаточно, чтобы получить dll с таской, которая умеет копировать файлы под мьютексом. Осталось только интегрировать.

## Берём ситуация в свои руки - интегрируемся в MSBuild
Для того, чтобы интегрировать таску с билд процесс, нужно её подключить к тестовому солюшену. В качестве прототипа и временного решения можно взять собранную SampleSolutionBuildTask.dll и положить в корень директории SampleSolution рядом. В Directory.Build.targets нужно добавить вызов таски:
```xml
<UsingTask
  TaskName="SampleSolutionBuildTask.CopyLocalWithMutex"
  AssemblyFile="SampleSolutionBuildTask.dll" />

  <Target
    Name="CopyLocalWithMutexTarget"
    AfterTargets="Build">
    <CopyLocalWithMutex
      OutputDirectoryPath="$(OutputPath)"
      ReferenceCopyLocalPaths="@(FilesForCopy)"
      CopyToOutputFromDependencies="@(TransitiveProjectsFiles)" />
  </Target>
```

Добавленный таргет будет вызываться после того, как закончится билд и будет вызывать созданную таску. Но подкладывать руками dll не очень хочется, вместо этого лучше использовать NuGet. Процесс создания подобного нюгета хорошо описан в статье Nate McMaster'а - https://natemcmaster.com/blog/2017/07/05/msbuild-task-in-nuget/. По его инструкции можно создать нюгет, который можно прописать в Directory.Build.props и это избавит от необходимости подкладывать dll самостоятельно:
```xml
<Project>
  <PropertyGroup>
    <ArtifactsPath>$(MSBuildThisFileDirectory)artifacts</ArtifactsPath>
    <UseCommonOutputDirectory>true</UseCommonOutputDirectory>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="SampleSolutionBuildTask" Version="1.0.0" />
  </ItemGroup>
</Project>
```

## Итог
MSBuild - это довольно хороший инструмент, который позволят долгое время не задумываться о том, как всё работает внутри. Если проблемы появились, то разобраться с этим возможно. Но примере проблемы с раздуванием Output директории было найдено пару не очевидных моментов, пару открытых issues, где есть проблемы. Об этом моментах нужно знать или узнать, чтобы решить поставленную задачу. Но решить её можно. MSBuid target'ы позволяет легко расширять билд процесс, добавить собственные шаги за небольшое количество строчек xml кода, который встраиваются прям в солюшен. С другой стороны MSBuild таски позволяют на C# написать собственную, более сложную, логику, которую хочется вставить в билд процесс. MSBuild в свою очередь предоставляет много полезного API - не нужно вручную обходить проекты, строить дерево зависимостей, искать подключённые пакеты. А возможность паковать MSBuild target'ы и task'и в нюгеты позволяет отделить вопрос билда в отдельный солюшен и разрабатывать независимым компонентом.
MSBuild - круто 😎