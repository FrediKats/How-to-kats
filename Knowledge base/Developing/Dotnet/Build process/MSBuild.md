---
title: MSBuild
---

MSBuild (The Microsoft Build Engine) - это платформа для сборки dotnet приложений.
Запустить сборку солюшена можно используя Visual Studio. Вместе с Visual Studio идёт msbuild.exe, который и отвечает за билд. Visual Studio использует .NET Framework реализацию MSBuild для загрузки и сборки проектов. Вместе с .NET Core появился альтернативный способ сборки - .NET Core реализация MSBuild, которая представлена CLI командами `dotnet build`. Новая функциональность, которую добавляют в MSBuild появляется и в Framework и в Core реализации, но .Framework содержит ряд API, которое изнчально не было перенесено в Core. Но это специфичные вещи (такие как использование COM объектов) и в большинстве случаев солюшен можно собрать как из Visual Studio, так и CLI командой.

## Solution restore
Основным [[MSBuild target|таргетом]] команды рестора является RestoreTask. Задача RestoreTask - сформировать список используемых [[Nuget]] пакетов в проектах и загрузить их. Есть два основных артефакта работы этой таски:
- [[MSBuild artifact directories#project.assets.json]]] файлы для каждого проекта
- Скачанные локально нюгеты в директории C:\\Users\\fredi\\.nuget\\packages\

Более подробно про логику резолва версий можно узнать с документации - https://learn.microsoft.com/en-us/nuget/concepts/dependency-resolution.

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

Нюгеты собираются под разные [[Dotnet frameworks|фреймворки]] для того, чтобы их можно было подключить к максимально большому количеству проектов. С точки зрения нюгетов основное отличие фреймворков - это набор доступного API. На сайте https://dotnet.microsoft.com/en-us/platform/dotnet-standard можно сравнить количество доступных методов в standard 2.0 и standard 2.1.

Но фреймворк - это не всегда приговор. Некоторое API backport'ят в старые версии с помощью нюгетов. Это в результате и создаёт разный набор зависимых нюгетов. Например, в Microsoft.Extensions.Logging для .NET Framework 4.6.2 есть такие зависимости:
- Microsoft.Bcl.AsyncInterfaces
- System.ValueTuple
- System.Diagnostics.DiagnosticSource

Но этих же зависимостей нет в .NET 8 потому что эти нюгеты являются частью стандартной конфигурации .NET 8.

Во время рестора фреймворк выбирается исходя из версии проекта, куда нужно нюгет подключить. Если проект версии 8.0.0, то среди доступных версий будет искаться версия для .NET 8, .NET 7 и так далее. Если на будет найдена версия для .NET Core, то будет искаться версия для .NET standard. Если не будет найдена и такая версия, то будет взята версия .NET Framework. Но такой рестор является не безопасным т.к. не всё API из Framework доступно в .NET 8 и рестор будет заканчиваться с предупреждениями. Проверить какая версия была выбрана можно в `project.assets.json`, там будет указан относительный путь к dll: `lib/net8.0/Microsoft.Extensions.Logging.dll`

Более подробно описано тут: https://learn.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks.
