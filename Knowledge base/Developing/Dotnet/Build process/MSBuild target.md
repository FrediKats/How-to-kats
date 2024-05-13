---
title: MSBuild target
---

[[MSBuild]] предоставляет возможность группировать элементы в таргеты, для описания более сложных сценариев. Пример таргета, который вызывается во время `dotnet restore`:
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

## Returns
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

Вызвать таргет и получить возвращаемое значение можно используя команду CallTarget:
```xml
<CallTarget Targets="_LoadRestoreGraphEntryPoints">
	<Output
		TaskParameter="RestoreGraphProjectInputItems"
		ItemName="VariableForReturnValues" />
</CallTarget>
```
