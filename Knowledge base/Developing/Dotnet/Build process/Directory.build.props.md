---
title: Directory.Build.props
---

Directory.build.props - это специальный файл [[MSBuild]]'а, который позволяет задавать шаренные настройки для [[Dotnet project]]. Созданный файл с таким названием в корневой директории солюшена будет влиять на настройки все проекты в этой директории. Синтаксис описания файла такой же, как и у csproj файла:
```xml
<Project>
	<PropertyGroup>
		<WarningLevel>999</WarningLevel>
		<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
	</PropertyGroup>
</Project>
```

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