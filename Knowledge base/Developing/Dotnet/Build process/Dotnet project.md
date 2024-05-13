---
title: .NET Project
---

Проект в [[Dotnet project system]] - это `.csproj` файл, который описывает единицу сборки.

## Project type ID
Все проекты в солюшене имеют определённый тип, этот тип описывается GUID'ом в [[Dotnet solution|.sln файле]]. Список известных ID есть на GitHub'е - https://github.com/JamesW75/visual-studio-project-type-guid.
## File format
 Существует два формата их описания. Первый формат был создан для .NET Framework. Этот формат можно опознать по объявлению Project ноды:

```
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
```

Этот формат считается устаревшим. И вместе с .NET Core был добавлен новый формат, который называют SDK-style:

```
<Project Sdk="Microsoft.NET.Sdk">
```

Новый формат позиционировался как замена, поэтому все они имел схожий набор возможностей. Но поведение у них отличалось. Например, в старом формате записи требуется явно указывать все файлы, которые нужно скомпилировать. А вместе с выходом SDK-style появилось и стало использоваться по умолчанию свойство [[Dotnet project#EnableDefaultItems|EnableDefaultItems]], которое автоматически добавляло все \*.cs файлы в проект и в процесс компиляции.
## Properties
Properties позволяют переопределять стандартные значения настроек проектов. Например, такая запись позволяет переопределить путь, куда будет сохраняться результат компиляции:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputPath>Output/</OutputPath>
  </PropertyGroup>
</Project>

```

### EnableDefaultItems
EnableDefaultItems позволяет регулировать автоматическое добавление лежащих в директоири файлов в проект.
- `true` (дефолтное значение в SDK-style проектах) добавляет в проект все файлы, которые находятся в директории проекта (за исключением [[MSBuild artifact directories#bin directory|bin]] и [[MSBuild artifact directories#obj directory|obj]] директорий).
- `false` (дефолтное значение для старого формата проектов) не добавляет автоматически файл. Это значит, что для того, чтобы файл с кодом попал в проект и компилировался, его нужно явно указать в .csproj файле.

## Items
Item'ы - это свойства, которые являются списком элементов, которые передаются билд системе для сборки. Обычно, это одно из:
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

