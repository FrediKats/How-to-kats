- [[2023-12-22-Development-feedback-loop|Developer feedback look]]
- Introduction to Roslyn analyzers using

---
## Roslyn the Compiler
### Compiler errors
[[Roslyn]] - это компилятор для языков C# и Visual Basic, который написан на C#. Языки программирования имеют чёткий синтаксис в соответствии с которым компиляторы могут его переводить в выполняемые файлы. Если написать код, который не соответствует синтаксису, то компиляторы генерируют ошибки. Обычно такие ошибки сопровождаются информативным сообщением, а интеграция с [[IDE]] разработчик получает редактор, который каждую проблему подсвечивает, сообщая, где именно и что не так. Например, если попытаться вызвать у класса несуществующий метод, то компилятор сообщить об ошибке, а IDE будет подсвечивать проблемное место:

```csharp
class MyClass
{
	public void MyMethod() { }
}

class OtherClass
{
	public void OtherMethod(MyClass myClass)
	{
		myClass.OtherMethod(); // CS1061	'MyClass' does not contain a definition
							   // for 'OtherMethod' and no accessible extension method 
							   // 'OtherMethod' accepting a first argument of type 
							   // 'MyClass' could be found (are you missing a using 
							   // directive or an assembly reference?)
	}
}
```

Все ошибки компиляции имеют свой идентификатор - CSxxxx. Например, CS0003 - это ошибка компиляции, которая говорит о том, что вовремя компиляции закончилась оперативная память. Описана эта информация в документации Microsoft - [Compiler Error CS0003 - C# | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/misc/cs0003).
### Compiler warnings
Roslyn умеет находить не только проблемы, которые блокируют процесс компиляции, но и ситуации, когда код можно скомпилировать, но с ним что-то не так. Примером такой ситуации является `CS0184 The given expression is never of the provided ('type') type`:
```csharp
class MyClass
{
	public static void Main()
	{
		int i = 0;
		if (i is string) // CS1084
			i++;
	}
}
```

С точки зрения синтаксиса языка код является корректным, но уже на стадии компиляции очевидно, что тело оператора if никогда не будет выполняться. Такие ошибки по умолчанию не превращаются в ошибки, а отображаются предупреждениями. Поведение этих предупреждений можно конфигурировать для проектов задавая в [[csproj file|.csproj-файле]]  значение `<WarningLevel>1</WarningLevel>`, где 1 - это уровень предупреждения:

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<OutputType>Exe</OutputType>
		<TargetFramework>net8.0</TargetFramework>
		<WarningLevel>999</WarningLevel>
	</PropertyGroup>
</Project>
```

Описание уровней можно найти в документации Microsoft - [C# Compiler Options - errors and warnings - C# | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/errors-warnings#warninglevel). Если кратко, то значение 0 выключает все опциональные проверки. Изначально были доступны значения 0-4, где 4 - отображать все возможные предупреждения. Но с каждой версией dotnet выходят новые проверки и для их включения нужно повышать WarningLevel. Например, вместе с C# 11 в dotnet добавили проверки, которые включаются на 7 уровне. Чтобы не повышать значение уровня после каждого нового релиза, можно указать значение 9999.

Ещё одна полезная опция, которую можно задать для предупреждений - это `TreatWarningsAsErrors`. Эта опция переводит все предупреждения в ошибки и появление любого предупреждения будет приводить к ошибке сборки проекта, даже если скомпилировать код возможно. Эта опция необходима для того, чтобы агрессивно пушить разработчиков к исправлению проблем.

## Configuration management in dotnet
### Directory.Build.props
Каждый проект с C# кодом представлен [[csproj file|csproj-файлом|]]. В этом файле сохраняется конфигурация проекта: используемая версия dotnet, уровень предупреждений компилятора  и многое другое. Но многие такие настройки концептуально хочется иметь на уровне всего [[Dotnet solution|солюшена]], а не прописывать в каждом проекта. Например, синхронизировать версию dotnet, чтобы обновлять в одно месте, а не в каждом проекте.

Готовым решением для унификации свойств проектов является [[Directory.build.props]]. Созданный файл с таким названием в корневой директории солюшена будет влиять на настройки все проекты в этой директории. Синтаксис описания файла такой же, как и у csproj файла. например, для выставления WarningLevel нужно создать такой Directory.Build.props:
```xml
<Project>
	<PropertyGroup>
		<WarningLevel>999</WarningLevel>
		<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
	</PropertyGroup>
</Project>
```
Если значение какого-то свойства задано только в Directory.Build.props, то оно будет применено к проекту. Если в csproj файле также задано значение свойству, то оно будет перезаписывать значение из Directory.Build.props.

Самый распространённый сценарий использования - это создание такого файла в корневой директории, но во время сборки проекта происходит поиск файла с таким названием по иерархии файловой системы. Например, если проект лежит в `c:\users\username\code\test\case1`, то файл будет искаться во всех этих директориях:
```
c:\users\username\code\test\case1
c:\users\username\code\test
c:\users\username\code
c:\users\username
c:\users
c:\
```

По умолчанию, поиск происходит до первого найденного файла Directory.Build.prop. Но в csproj файле можно прописать `<Import>`, который будет явно добавлять дополнительные Directory.Build.props файлы к проекту.

### .editorconfig
[[editorconfig|.editorconfig]] - это специальный формат ini-файла, который используется для описания настроек проекта, в том числе кодстайла. Он не привязан к конкретному языку и является универсальным инструментом.

Чтобы подключить к C# проекту .editorconfig его достаточно положить в директорию солюшена.
```
SolutionDirectory/
  Project/**
  MySolution.sln
  .editorconfig
```
Чтобы убедиться, что процесс сборки проекта учитывает .editorconfig файл, можно выполнить консольную команду сборки с повышенным логированием: `dotnet build --verbosity normal` и в логах сборки найти строчки `/analyzerconfig:...`. Одна из таких строчек будет содержать путь к .editorconfig'у в корне солюшена.

Сам файл представляет собой ini файл с парами ключ-значение. На сайте .editorconfig'а ([EditorConfig](https://editorconfig.org/#file-format-details)) можно найти такие примеры:
```ini
[*]
end_of_line = crlf
insert_final_newline = true
indent_style = tab
```
По спецификации ini файлов, помимо строчек с парами ключ-значение, поддерживаются также категории. Для .editorconfig эти категории используются для того, чтобы определить скоуп применения правил. Выглядит это так:
```ini
[*.{cs,vb}]
end_of_line = crlf

[*.cs]
indent_style = spaces
indent_size = 4
```
Если написать все опции без этих категорий, то они просто не будут применяться.

### Severity
Roslyn анализаторы имеют Severity, который влияет на то, как будут интерпретироваться найденные диагностики. Диагностики - это те ошибки и предупреждения, которые генерируются в Error list'е в IDE. Severity анализатора определяет будут ли они ошибками, предупреждениями или чем-то ещё. Виды Severity:
- error
- warning
- suggestion
- silent
- none

Например, если выставлен `WarningLevel = 4`, то все более новые проверки будут с `severity = none`. Если выставить `TreatWarningsAsErrors = true`, то все диагностики будут иметь `severity = error`.

Для более гибкой конфигурации severity можно использовать .editorconfig. Конфигурация имеет такой синтаксис:
```ini
[*.cs]
dotnet_diagnostic.CS8981.severity = error
```

Такая запись означает, что для правила CS8981 будет применяться не стандартный уровень severity, а `error`. Используя все описанные инструменты можно:
- Включить все опциональные проверки для всех проектов
- Зафорсить, чтобы все предупреждения должны быть ошибками
- Точечно в .editorconfig выключить несколько проверок, которые не являются важными для проекта

## Roslyn the analyzer
### Compiler as a Service
Одна из главных идеологических особенностей Roslyn'а - это позиционирование как Compiler as a Service. Roslyn раскрыл многие аспекты работы компилятора как API и позволил встраиваться в его pipeline. Одна из возможностей, которая в результате появилась - это возможность использовать информацию про синтаксис и семантику кода для написания анализаторов кода. Анализатор (в контексте Roslyn) - это код, который реализует предоставленное API Roslyn'а, выполняет анализ кода и генерирует диагностики.

Есть несколько способов подключения анализаторов к проекту:
- Включение встроенных в дотнет анализаторов. Они появились и идут вместе с дотнетом начиная с .NET 5. Для .NET Framework нужно устанавливать `Microsoft.CodeAnalysis.NetAnalyzers`
- Подключение [[Nuget]] пакетов с анализаторами
- Установка расширения на IDE с анализатором

Все встроенные правила можно выделить в две категории - Style rules (начинаются на IDE) и Quality rules (начинаются на CA). Основным публичным источником знаний о таких анализаторах является MS Learn ([Code analysis rule categories - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/categories)), а точнее GitHub репозиторий и из которого страницы на MS Learn генерируются ([docs/docs/fundamentals/code-analysis at main · dotnet/docs (github.com)](https://github.com/dotnet/docs/tree/main/docs/fundamentals/code-analysis)). 
### Quality rule analyzers
Quality rule analyzers нацелены на поиск проблем в коде, работают схожим образом с CSxxxx проверками. Для включения CAxxxx анализаторов необходимо выставить опцию `<EnableNETAnalyzers>true</EnableNETAnalyzers>`.

Рассмотрим структуру CA-правил на примере [CA1021](https://github.com/dotnet/docs/blob/main/docs/fundamentals/code-analysis/quality-rules/ca1021.md):
- ID - CA1021
- Название - Avoid out parameters
- Описание - Passing types by reference (using `out` or `ref`) requires experience with pointers, understanding how value types and reference types differ, and handling methods with multiple return values. Also, the difference between `out` and `ref` parameters is not widely understood...
Используя .editorconfig и зная ID правила можно сконфигурировать его severity или выключить для отдельного проекта:
```
dotnet_diagnostic.CA1021.severity = none # can be replaces with 'error'
```

Некоторые CAxxxx анализаторы предоставляют возможность для более гибкой настройки. Например, CA1062 требует проверять на null все аргументы публичных методов. Под проверкой на null подразумевается любой встроенный метод проверки: `if (value is null)` или `ArgumentNullException.ThrowIfNull`. Но если по какой-то причине нужно использовать другие методы проверки (например, в кодовой базе были написаны собственные методы до того, как появился метод `ArgumentNullException.ThrowIfNull`), то вместе с правилом можно указать дополнительные методы, которые будут трактоваться как проверки на null:
```ini
## Validate arguments of public methods (CA1062)
dotnet_diagnostic.CA1062.severity = warning
dotnet_code_quality.CA1062.null_check_validation_methods = ThrowIfNull
```
### Style rule analyzers
Style rule analyzers - это анализаторы, которые проверяют соответствие сконфигурированному стилю кода - начиная от расположения пробелов и заканчивая запретом использования фичей. Полный список анализаторов описан на сайте  с документацией - [Code-style rules overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/). В. Для включения IDExxxx анализаторов необходимо выставить опцию `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>`.

Рассмотрим структуру IDE-правил на примере IDE0011 ([docs/docs/fundamentals/code-analysis/style-rules/ide0011.md at main · dotnet/docs (github.com)](https://github.com/dotnet/docs/blob/main/docs/fundamentals/code-analysis/style-rules/ide0011.md)):
- Идентификатор правила - IDE0011
- Название правила - Add braces
- Краткое описание правила - This style rule concerns the use of curly braces `{ }` to surround code blocks.
- Список опций, которые конфигурируют работу правила - csharp_prefer_braces.
Отличии от CSxxxx правил (исключения есть), у IDExxxx правил часто бывают опции, которые позволяют задавать что именно ожидается от проверки. В случае IDE011, правило проверяет, что скобки выставлены корректно. Если они выставлены не корретно - кидается диагностика. А опция этого правила csharp_prefer_braces задаёт что именно является "корректным" в данном проекте. И в .editorconfig'е будет две строчки:
```
dotnet_diagnostic.IDE0011.severity = warning
csharp_prefer_braces = when_multiline
```

Среди IDE правил есть "парные правила". Например, IDE0007 и IDE0008. Эта правила регулируют использование keyword'а `var`. IDE0007 кидает диагностики, если был использован тип в том месте, где по кодстайлу ожидается var, а IDE0008 наоборот кидает диагностики, если были использован var там, где ожидается явное указание типа. Эти два правила имеют шаренный список опций, которые и определяют где же ожидается var, а где не ожидается (csharp_style_var_for_built_in_types, csharp_style_var_when_type_is_apparent, csharp_style_var_elsewhere).

Очень особенным правилом является IDE0055. IDE0055 - это Formatting rule, общее правило выставления для регулирования пробелов, переносов строк и подобного. Под этим правилом находится большая часть существующих правил. Они описаны в двух местах: [docs/docs/fundamentals/code-analysis/style-rules/dotnet-formatting-options.md at main · dotnet/docs (github.com)](https://github.com/dotnet/docs/blob/main/docs/fundamentals/code-analysis/style-rules/dotnet-formatting-options.md) и [docs/docs/fundamentals/code-analysis/style-rules/csharp-formatting-options.md at main · dotnet/docs (github.com)](https://github.com/dotnet/docs/blob/main/docs/fundamentals/code-analysis/style-rules/csharp-formatting-options.md). Например, чтобы зафорсить написание скобочек if'а с новой строки нужно включить IDE и выставить опцию csharp_new_line_before_open_brace:
```
dotnet_diagnostic.IDE0055.severity = warning
csharp_new_line_before_open_brace = all
```
И соответственно, если выставить опцию, но не зафорсить IDE0055, то диагностики не будут форсится. Если выставить severity для IDE0055, но не прописать все правила, то диагностики будут появляться, но они будут ссылаться на стандартные настройки кодстайла, которые могут не подходить под конкретный проект.

### IDE0055 и crlf/lf
Привило IDE0055 помимо очевидных вещей умеет проверять символы новой строки. Задать ожидаемый способ переноса можно в .editoconfig:
```ini
end_of_line = crlf
```

Если очень сильно упростить, то есть два разных подхода к указанию символа переноса строки:
- Windows использует '\\r\\n'
- Unix использует '\\n'

И этого достаточно, чтобы превратить CI в ад. Если для написания кода используется Windows, а для выполнения CI - Linux, то может получиться ситуация, когда на Windows солюшен собирается, а на Linux падает с error IDE0055: Fix formatting.
При попадании '\\n' в Windows может происходить много неожиданных артефактов. Visual Studio при открытии файлов с неправильными переносами будет сообщать о проблеме и предлагать исправить переносы. 

Довольно подробно тема описана на SO - [newline - How line ending conversions work with git core.autocrlf between different operating systems - Stack Overflow](https://stackoverflow.com/questions/3206843/how-line-ending-conversions-work-with-git-core-autocrlf-between-different-operat). Если кратко, то решение сводится к использованию [[git#autocrlf]]:
- На Windows нужно устанавливать `git config --global core.autocrlf true`.  Эта опция при доставании изменения из git использует в качестве EOL crlf, а перед внесением изменений заменяет EOL на lf
- На non-Windows нужно ставить `git config --global core.autocrlf input`. Эта опиция при доставании изменений не меняет EOL, а при внесении заменяет EOL на lf, если где-то случайно вставился crlf
Такая конфигурация позволит в git всегда хранить lf, а на Windows автоматически использовать crlf, чтобы не создавать проблемы. Из этого также следует, что опции `end_of_line = crlf` или `end_of_line = lf` работать не будут.
### BannedApiAnalyzers
Помимо стандартных анализаторов, которые идут вместе с dotnet, существует много других нюгет. Они более ситуативные, но для них есть применение. Например, анализатор `Microsoft.CodeAnalysis.BannedApiAnalyzers` позволяет указать API, которое будет запрещено к использованию.

Пример использования: нужно разработать приложение, которое будет взаимодействовать со временем. Для написания тестов, скорее всего, понадобиться уметь устанавливать нужное время, перематывать его. Для этого нужно по всему коду отказаться от DateTime.Now в пользу `ITimeProvider`. В этой ситуацию можно прописать в BannedAPI все статические методы. которые возвращают время и быть уверенным, что в коде будет использоваться только `ITimeProvider`.
### Минифицированный способ записи в .editorcofing 
Иногда можно встретить альтернативную запись, где смешивание значение опции и severity - `option_name = value:severity`:
```
[*.cs]
dotnet_style_qualification_for_field = true:warning
```
Например, сейчас именно в таком формате генерируются отсутствующие опции, если открыть .editorconfig в Visual Studio. И такой формат даже работает, но есть два нюанса:
- Работает только там, где работает. Некоторые опции не поддерживают такое указание, выяснить какие именно поддерживают сложно.
- Уже с давних времён (2020-ого года) есть план отказаться от поддержки этого формата в пользу декомпозиции на две отдельные строки. Более подробно тут - [Deprecate 'severity' field for IDE code style editorconfig syntax · Issue #44201 · dotnet/roslyn · GitHub](https://github.com/dotnet/roslyn/issues/44201).

Знание о том, как он работает, полезно, но лучше его не использовать.
### dotnet format
dotnet format - это CLI, который позволяет упростить работу с анализаторами. Команда `dotnet format solution-path` будет искать все ошибки в коде в соответствии с настроенными анализаторами и применять исправления для них. Другое использование - это валидация без исправлений, которую можно выполнить добавив соответствующий ключ: `dotnet format solution-path --verify-no-changes`. В большинстве случаев все анализаторы можно настроить и встроить в билд и отдельно выполнять запуск dotnet format не нужно. Есть редкие исключения, например опция `charset` может быть найдена исправлена и исправлена dotnet format, но не может быть найдена анализатором из-за того, что анализаторы работают с содержимым файла.

## Конфигурация Roslyn анализаторов
### Настройка .editorconfig'а
Настройка CAxxxx и IDExxxx правил отличается.
IDE анализаторы имеют опции, которые задают ожидаемое оформление кода. Если включить EnforceCodeStyleInBuild и ничего не конфигурировать в .editorconfig, то IDE все анализаторы будут требовать оформления в соответствии со стандартными настройками. Чтобы подстроить под себя анализаторы, нужно все ожидаемые значения для всех опций.

CAxxxx анализаторы включаются опцией `EnableNETAnalyzers`. Но выставление этой опции не включается все анализаторы сразу. Набор включаемых анализаторов регулируется опцией AnalysisLevel, которая работает схожим образом с WarningLevel. Более подробно описано в документации - [MSBuild properties for Microsoft.NET.Sdk - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/project-sdk/msbuild-props#analysislevel). Но есть альтернативный способ конфигурации - включать необходимые анализаторы используя .editorconfig. Если не выставлен AnalysisLevel, то все анализаторы будут выключены.
### Well-known issues: GenerateDocumentationFile
Всю историю можно прочитать тут: [Compiler does not report unused imports when XML doc comments are disabled · Issue #41640 · dotnet/roslyn · GitHub](https://github.com/dotnet/roslyn/issues/41640).

Если кратко, то существует баг из-за которого не работало правило IDE0005. Оказалось, что оно не работает, если не добавить в .csproj/.props опцию `<GenerateDocumentationFile>true</GenerateDocumentationFile`. Это потянет за собой срабатывания правила CS1591, которое репортит диагностики о том, что в public API есть метод без xml документации. Поэтому бонусом нужно ещё и отключить эти warning'и:
```xml
<Project>
    <PropertyGroup>
        <NoWarn>$(NoWarn);CS1591</NoWarn>
        <GenerateDocumentationFile>true</GenerateDocumentationFile>
    </PropertyGroup>
</Project>
```

## Интеграция .editorconfig и Visual Studio
.editorconfig можно открыть в Visual Studio разными способами. Если добавить .editorconfig как файл проекта и попытаться его открыть, то от будет открываться в стандартном ini редакторе. Но если попытаться добавить в солюшен как `Add > Existing item` и потом открыть, то он будет открываться в специальном UI для редактирования .editorconfig.

Этот UI довольно неплохо подходит для того, чтобы не вникая глубоко растыкать чекбоксы и получить приемлемый .editorconfig. Чекбоксы изменяют .editorconfig и все изменения можно увидеть в нём. Но у этого варианта есть одна особенность. Visual studio модифицирует .editorconfig и добавляет туда то, что посчитает нужным. Например, если в нём не описана опция, которая по мнению VS должна быть описана, то после открытия этого UI редактора она будет сгенерирована со стандартными значениями. Вишенкой на торте являются экспериментальные опции (dotnet_style_allow_statement_immediately_after_block_experimental), которые также добавляются, но они не отображаются в UI, не задокументированы, и не планируются документироваться (https://github.com/dotnet/roslyn/issues/60539#issuecomment-1086707135). Поэтому основная рекомендация при работе с UI - закомитить .editorconfig до открытия в VS, чтобы в git можно было отследить изменения после открытия.

### Итоговая минимальная настройка Directory.Build.props
После добавления всех необходимых опций для анализаторов должен получиться такой Directory.Build.props:

```xml
<Project>
	<PropertyGroup>
		<WarningLevel>999</WarningLevel>
		<EnableNETAnalyzers>true</EnableNETAnalyzers>
		<!-- CA rule configuration located in .editorconfig -->
		<!-- <AnalysisLevel>latest</AnalysisLevel> -->
		<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
		<NoWarn>$(NoWarn);CS1591</NoWarn>
		<GenerateDocumentationFile>true</GenerateDocumentationFile>
		<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
	</PropertyGroup>
</Project>

```

### Шаринг настроек между солюшенами
Как только проблема в настройкой решается в одном проекте, созданный .editorconfig файл начинает копироваться из проекта в проект. И всё хорошо пока не понадобиться внести изменения в этот .editorconfig. Ввиду того, что файлы копировались, очень сложно отследить их актуальность. Из вариантов - писать комментариями в начале версию и дату изменения. Но звучит не надёжно.

Стандартный механизм шаринга кода между процессами в dotnet - это Nuget. Но на данный момент нет готового решения. В репозитории Roslyn'а есть issue на эту тему - [Define spec for NuGet packages providing .editorconfig defaults · Issue #19028 · dotnet/roslyn · GitHub](https://github.com/dotnet/roslyn/issues/19028).

Но при должной подготовке и желании, реализовать доставку конфигурации через nuget можно самостоятельно.

Шаг 1. Создать C# library проект, который будет создавать Nuget. Файл `MyProject.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <GeneratePackageOnBuild>True</GeneratePackageOnBuild>
  </PropertyGroup>
</Project>
```

Шаг 2. Создать в директории проекта файл `MyProject.props`, который будет содержать необходимые шаренные настройки:
```xml
<Project>
	<PropertyGroup>
		<WarningLevel>999</WarningLevel>
		<EnableNETAnalyzers>true</EnableNETAnalyzers>
		<!-- CA rule configuration located in .editorconfig -->
		<!-- <AnalysisLevel>latest</AnalysisLevel> -->
		<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
		<NoWarn>$(NoWarn);CS1591</NoWarn>
		<GenerateDocumentationFile>true</GenerateDocumentationFile>
		<TreatWarningsAsErrors>true</TreatWarningsAsErrors>

		<Nullable>enable</Nullable>
		<SuppressNETCoreSdkPreviewMessage>true</SuppressNETCoreSdkPreviewMessage>
	</PropertyGroup>
</Project>
```

Шаг 3. Создать в директории проекта файл .editorconfig

Шаг 4. Добавить файлы в процесс запаковки nuget'а. По умолчанию, файлы из проекта не пакуются в nuget, нужно в .csproj прописать явное добавление. Для запаковки файлов, которые начинаются с точки нужно прописать опцию `NoDefaultExcludes`. Более подробно можно прочитать в документации ([NuGet CLI pack command | Microsoft Learn](https://learn.microsoft.com/en-us/nuget/reference/cli-reference/cli-ref-pack)). При указании путей стоит использовать '\\'. чтобы пути корректно работали в Windows и Linux системах.
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <NoDefaultExcludes>true</NoDefaultExcludes>
    <GeneratePackageOnBuild>True</GeneratePackageOnBuild>
  </PropertyGroup>

  <ItemGroup>
    <None Include="ProjectName.props" Pack="true" PackagePath="\build" />
    <None Include=".editorconfig" Pack="true" PackagePath="\content\Rules" />
  </ItemGroup>
</Project>
```

Шаг 5. Описать логику копирования (в будущем напишу более подробно о том, что такое Target'ы) .editorconfig файла во время сборки проекта в .props файле:
```xml
<Project>
    <PropertyGroup>
		<!-- ... -->
    </PropertyGroup>
    <ItemGroup>
		<EditorConfigFilesToCopy Include="$(MSBuildThisFileDirectory)..\content\Rules\.editorconfig" />
	</ItemGroup>

    <Target Name="CopyEditorConfig" BeforeTargets="BeforeBuild">
        <Message Text="Copying the .editorconfig file from '@(EditorConfigFilesToCopy)' to '$(SolutionDir)'" />
        <Copy
            SourceFiles="@(EditorConfigFilesToCopy)"
            DestinationFolder="$(SolutionDir)"
            SkipUnchangedFiles="true"
            UseHardlinksIfPossible="false" />
    </Target>
</Project>
```

После всех шагов проект можно собрать в нюгет и добавить в другой солюшен. После добавления нюгета при первой сборке будет выполнять копирование .editorconfig файла, а .props файл будет добавлен автоматически.

Следующий шаг, который можно сделать - это добавление .editorconfig'а в .gitignore. Файл .editorconfig'а является генерируемым и чтобы не забивать историю изменений, можно его исключить.
