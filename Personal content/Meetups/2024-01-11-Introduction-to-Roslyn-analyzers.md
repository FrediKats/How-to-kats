### Цикл фидбека

Цикл фидбека - это процесс через который проходит фидбек, чтобы добраться до разработчика.

- Разработчик -> Клиент
- Разработчик -> Тестировщик
- Разработчик -> Интеграционные тесты
- Разработчик -> CI
- Разработчик -> Компилятор

---
### Roslyn the Compiler
Roslyn - это компилятор для языков C# и Visual Basic. Как и все компиляторы, он умеет сообщать об ошибках.


```csharp
class MyClass
{
  public void MyMethod() { }
}

MyClass myClass = ...;
myClass.OtherMethod(); // CS1061
  // 'MyClass' does not contain a definition
  // for 'OtherMethod' and no accessible extension method
  // 'OtherMethod' accepting a first argument of type
  // 'MyClass' could be found (are you missing a using
  // directive or an assembly reference?)
```

---
### Compiler errors
Все ошибки компилятора имеют идентификатор - CSxxxx. Например, CS0003 - это ошибка компиляции, которая говорит о том, что вовремя компиляции закончилась оперативная память.

Найти их можно на learn'е: learn.microsoft.com/en-us/dotnet/csharp/misc/cs0003

---
### Compiler warnings
Roslyn умеет находить не только проблемы, которые блокируют процесс компиляции, но и ситуации, когда код можно скомпилировать, но с ним что-то не так.

```csharp
int i = 0;
// CS0184 The given expression
// is never of the provided ('type') type
if (i is string)
  i++;
```
  
---
### Compiler warnings configuration
Предупреждения конфигурируются через свойство проекта **WarningLevel**.
  
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <WarningLevel>999</WarningLevel>
  </PropertyGroup>
</Project>
```
  
---
### Treat Warnings As Errors
Предупреждения не блокируют билд. Но это также конфигурируется. **TreatWarningsAsErrors** превращает все предупреждения в ошибки.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <WarningLevel>999</WarningLevel>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```
  
---
### Directory.Build.props
Directory.Build.props - это специальный файл MSBuild'а, который позволяет вынести общие для солюшена настройки из проектов.
  
Пример такого файла:
```xml
<Project>
  <PropertyGroup>
  <WarningLevel>999</WarningLevel>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```
  
---
### Расположение props файла
Обычно такой файл располагается в корне солюшена, но MSBuild ищет по всей иерархии. Например, если проект лежит в `c:/users/username/code/test/case1`, то файл будет искаться во всех этих директориях:
```
c:/users/username/code/test/case1
c:/users/username/code/test
c:/users/username/code
c:/users/username
c:/users
c:/
```
  
---
  
> [!warning] Ради всего святого, не называйте директории Directory.Build.props или .editorconfig
  
---
### .editorconfig
Специальный формат ini-файла, который используется для описания настроек проекта, в том числе кодстайла. Он не привязан к конкретному языку и является универсальным инструментом.
  
```ini
[*]
end_of_line = crlf
insert_final_newline = false
indent_style = tab
```
---
### Указание расширения для настроек
В .editorconfig файле важно указывать для каких файлов нужно применить настройки:
 
```ini
[*.{cs,vb}]
end_of_line = crlf
  
[*.cs]
indent_style = spaces
indent_size = 4
  
[*]
insert_final_newline = false
```
---
### Severity
У Roslyn'а есть несколько уровней severity: error, warning, suggestion, silent, none.
  
`WarningLevel = 1` значит, что для всех проверок уровня 2 и выше будет выставлен `severity = none`.
  
`TreatWarningsAsErrors = true` значит, что для всех проверок вместо `warning` будет использоваться `error`.
  
---
### Severity via .editorconfig
Конфигурация severity через .editorconfig:
```ini
[*.cs]
dotnet_diagnostic.CS8981.severity = error
```
  
Таким образом можно:
- Включить все опциональные проверки
- Зафорсить, чтобы все предупреждения должны быть ошибками
- Точечно в .editorconfig выключить несколько проверок, которые не являются важными для проекта
  
---
### Compiler as a Service
Roslyn - Compiler as a Service. Roslyn, он предоставляет API для интеграции в процессе сборки. Одна из реализаций такой интеграции - это анализаторы.
  
- Ряд анализаторов поставляются вместе с .NET 5+
- Для старых версий можно подключить нюгетом: `Microsoft.CodeAnalysis.NetAnalyzers`
- В VS анализаторы можно добавить как расширение на IDE
  
---
### Default Roslyn analyzers
Все встроенные правила можно выделить в две категории:
- Style rules (IDExxxx)
- Quality rules (CAxxxx)
  
Основным публичным источником знаний о таких анализаторах является MS Learn, но весь контент туда генерируется из GitHub репозитория github.com/dotnet/docs.
  
---
### Quality rules
Рассмотрим структуру CA-правил на примере CA1021:
- ID - CA1021
- Название - Avoid out parameters
- Описание - Passing types by reference (using `out` or `ref`) requires experience with pointers, understanding how value types and reference types differ, and handling methods with multiple return values. Also, the difference between `out` and `ref` parameters is not widely understood...
  
---
### Style rules
Style rule analyzers - это анализаторы, которые проверяют соответствие сконфигурированному стилю кода - начиная от расположения пробелов и заканчивая запретом использования фичей.
  
Примере IDE0011:
- Идентификатор правила - IDE0011
- Название правила - Add braces
- Краткое описание правила - This style rule concerns the use of curly braces `{ }` to surround code blocks.
- Список опций, которые конфигурируют работу правила - csharp_prefer_braces.
 
---
### Style rule options
Большинство IDE правил имеют опции, которые меняют ожидание анализатора. Для IDE0011 есть опция csharp_prefer_braces, которая позволяет задавать ожидаемое расположение скобок.
  
В .editorconfig'е будет две строчки:
```
dotnet_diagnostic.IDE0011.severity = warning
csharp_prefer_braces = when_multiline
```
  
---
### IDE0055
IDE0055 - это Formatting rule, общее правило выставления для регулирования пробелов, переносов строк и подобного.
  
Пример конфигурации, которая форсит перенос скобок на новую строку:
```
dotnet_diagnostic.IDE0055.severity = warning
csharp_new_line_before_open_brace = all
```
  
---
### IDE0055 и crlf/lf
IDE0055 также регулирует символ конца строки. Задать его можно в .editoconfig:
```ini
end_of_line = crlf
```
  
Если использовать Windows и Linux вместе, то переносы могут сильно ломать diff. Единственное нормальное решение - это git autocrlf:
- Windows: `git config --global core.autocrlf true`
- non-Windows: `git config --global core.autocrlf input`
  
---
### CA1062
CA1062 анализатор требует проверять все используемые аргументы публичных методов на null. Под проверкой на null подразумевается любой встроенный метод проверки: `if (value is null)` или `ArgumentNullException.ThrowIfNull`.
  
Анализатор позволяет задавать собственные методы, которые будут считаться проверками на null:
  
```ini
dotnet_diagnostic.CA1062.severity
  = warning
  
dotnet_diagnostic.CA1062.null_check_validation_methods
  = ThrowIfNull
```
  
---
### Well-known issues: GenerateDocumentationFile
Всю историю можно прочитать тут: https://github.com/dotnet/roslyn/issues/41640.
  
Бага в анализаторе IDE0005 требует выставления GenerateDocumentationFile. А это свойство генерирует ошибки CS1591.
  
Чтобы это всё работало нужно:
```xml
<PropertyGroup>
  <NoWarn>$(NoWarn);CS1591</NoWarn>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>
```
  
---
### BannedApiAnalyzers
Microsoft.CodeAnalysis.BannedApiAnalyzers позволяет указать API, которое будет запрещено к использованию.
  
Пример использования: для того, чтобы код был тестируемым, хочется запретить использовать DateTime.Now.
  
---
### System.IO.Abstractions.Analyzers
Часто анализаторы пишут под конкретные библиотеки. Например, System.IO.Abstractions - это Nuget с абстракциями для работы с файлами.
  
System.IO.Abstractions.Analyzers - это Nuget с анализаторами, которые проверяют, что в коде не используется логика работы с FS в обход абстракциям.
  
---
### Code fixers
Некоторые анализаторы поставляются вместе с фиксерами. Code fixer - это автоматические изменения, которые исправляют проблему, на которую указывает анализатор.

IDE на диагностики добавляют в контекстное меню команду запуска код фиксера. Некоторые код фиксеры можно выполнить сразу на весь файл, проект, солюшен.
  
---
### dotnet format
dotnet format - это CLI, который позволяет упростить работу с анализаторами.
  
`dotnet format solution-path` - найти все диагностики и применить доступные кодфиксеры.
  
`dotnet format solution-path --verify-no-changes` - проверить наличие диагностик.
  
Компилятор и dotnet format генерируют почти всегда одинаковый результат.
Есть редкие исключения, например опция `charset`.
  
---
### Настройка Directory.Build.props и .editorconfig
Настройка IDExxxx - нужно выставить `EnforceCodeStyleInBuild` и сконфигурировать все опции, чтобы форматирование давало ожидаемый результат.
  
Настройка CAxxxx - нужно выставить `EnableNETAnalyzers` и включить нужный AnalysisLevel либо включить нужные анализаторы.
  
---
### Минифицированный способ записи в .editorcofing
Минифицированный способ записи - это смешивание значений опции и severity:
```ini
[*.cs]
dotnet_style_qualification_for_field = true:warning
```
Нюанса:
- Работает только там, где работает.
- С 2020-ого года ходят убрать поддержку такого формата - https://github.com/dotnet/roslyn/issues/44201.

---
## Интеграция .editorconfig и Visual Studio
Visual Studio имеет собственный UI редактор .editorconfig файлов.
  
Плюс: Хорошо подходит для того, чтобы не вникая глубоко растыкать чекбоксы/
  
Минус: Visual studio модифицирует .editorconfig и добавляет туда то, что посчитает нужным.
  
Рекомендация: закомитить .editorconfig до открытия в VS, чтобы в git можно было отследить изменения после открытия.
  
---
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

---
### Шаринг настроек между солюшенами
  
Проблема: Чтобы сконфигурировать все анализаторы нужно создать .editorconfig и Directory.Build.props. Чтобы переиспользовать в другом солюшене, эти файлы нужно копировать. Поддерживать версионирование и синхронизировать тяжело.
  
---
### Шаринг настроек между солюшенами: Nuget?
Краткий ответ: Нет.
  
Стандартный механизм шаринга кода между процессами в dotnet - это Nuget. Но на данный момент нет готового решения. В репозитории Roslyn'а есть issue на эту тему - https://github.com/dotnet/roslyn/issues/19028.
  
---
### Шаринг настроек между солюшенами: Nuget!

Не краткий ответ: Нет, но если очень хочется...
  
При должной подготовке и желании, реализовать доставку конфигурации через nuget можно самостоятельно. Заодно узнать много нового про нюгеты.
  
---
### Шаг 1. Создание проекта
Нужно создать обычный проект, который будет паковаться в нюгет. Файл `MyProject.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <GeneratePackageOnBuild>True</GeneratePackageOnBuild>
  </PropertyGroup>
</Project>
```

---
### Шаг 2. Добавить .editorconfig и props
Нюгет должен распространять два файла, которые нужно добавить в проект - `.editorconfig` и `MyProject.props`.

```xml
<Project>
  <PropertyGroup>
    <WarningLevel>999</WarningLevel>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <NoWarn>$(NoWarn);CS1591</NoWarn>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```
  
---
### Шаг 3. Добавление файлов в нюгет
- По умолчанию, файлы из проекта не пакуются в nuget, нужно в .csproj прописать явное добавление
- Для запаковки файлов, которые начинаются с точки нужно прописать опцию `NoDefaultExcludes`
  
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
 
---
### Шаг 4. Описать логику распаковки
Во время сборки файл .editorconfig должен попасть в корень солюшена. Логику его копирования должна быть описана в .props файле:
```xml
<Project>
  <PropertyGroup>
    <!-- ... -->
  </PropertyGroup>
  <ItemGroup>
    <EditorConfigFilesToCopy
    Include="$(MSBuildThisFileDirectory)..\content\Rules\.editorconfig" />
  </ItemGroup>
  
  <Target Name="CopyEditorConfig" BeforeTargets="BeforeBuild">
    <Message
      Text="Copying the .editorconfig file from '@(EditorConfigFilesToCopy)' to '$(SolutionDir)'" />
     <Copy
       SourceFiles="@(EditorConfigFilesToCopy)"
       DestinationFolder="$(SolutionDir)"
       SkipUnchangedFiles="true"
       UseHardlinksIfPossible="false" />
  </Target>
</Project>
```

---
### Шаг 5. Profit!!1!
Нюгет готов к добавлению в другие проект. После добавления нюгета при первой сборке будет выполнять копирование .editorconfig файла, а .props файл будет использован автоматически.

Опционально можно добавить .editorconfig в .gitignore т.к. это теперь генерируемый файл.

---
### Summary

Анализаторы - круто.