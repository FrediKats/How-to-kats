---
title: Solution metaproject
---
[[MSBuild]] во время сборки оперирует [[Dotnet project|проектами]]. При сборке [[Dotnet solution|solution'а]] создаётся искусственный проект `SolutionName.csproj.metaproj`, на котором и вызывается собка.