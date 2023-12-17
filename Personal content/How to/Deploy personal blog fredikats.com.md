FrediKats.com - это персональный блог, который собирается из [[Markdown]] файлов. За основу взят шаблон [yegor256/bloghacks: Jekyll demo blog for "256 Bloghacks" book (github.com)](https://github.com/yegor256/bloghacks) от [[Yegor Bugayenko|Егора Бугаенко]].

## Алгоритм локального развёртывания сайта
- Установить [[Docker]], например через [[winget]]: `winget install Docker.DockerDesktop`
- Скопировать исходный шаблон [yegor256/bloghacks: Jekyll demo blog for "256 Bloghacks" book (github.com)](https://github.com/yegor256/bloghacks)
- Открывать [[PowerShell]], перейти в директорию, куда был скопирован шаблон
- Выполнить команду `docker run -it --rm -v "${PWD}:/b" -p 4000:4000 yegor256/blog-image 'cd /b && bundle update && bundle exec jekyll serve --trace --future --host=0.0.0.0'`
	- `docker run` создаст и запустит контейнер
	- `-v "${PWD}:/b"` добавляет [[Docker#Volume]], чтобы докеру был доступен шаблон. `${PWD}` - это путь к текущей директории. Вместо этого можно указать полный путь директории, где расположен шаблон. `/b` - путь куда в докере будет монтироваться директория
	- `yegor256/blog-image` - указание [[Docker image|образа]], который будет использован для поднятия контейнера. Это специальный образ, который был собран [[Yegor Bugayenko|Егором]] для этих целей.
	- `cd ...` - команды, которые передаются контейнеру для обработки
		- `cd /b` - переход в директорию с шаблоном. Нужно синхронизировать с путём монтирования [[Docker#Volume]]
		- `exec jekyll serve` запускает [[Jekyll]] сервер  
- Открыть http://localhost:4000/, убедиться, что сайт доступен

## Добавление нового поста

Посты лежат в директории `_posts/`. Для добавление нового поста нужно создать `.md` файл, например `_posts/2023/2023-12-15-My-new-post.md`. Пост должен иметь хедер:
```
---
layout: post
date: 2023-12-15
title: "My new post"
---
```

На главной странице сайта отображается лента из постов. Шаблон использует [[Jekyll#excerpt]] для отображения превью постов. Чтобы обозначить, какую часть поста нужно показывать, нужно использовать `<!--more-->` (задаётся в `_config.yml`).

Для добавления изображений на страницы в оригинальном блоге ([yegor256/blog: My blog about computers, written in Jekyll and deployed to GitHub Pages](https://github.com/yegor256/blog)) используется самописный плагин: https://github.com/yegor256/blog/blob/master/_plugins/pictures.rb. Он позволяет в постах не писать \<image\> теги, а использовать [[Jekyll#Liquid Template|Liquid Template]]: `{% picture /images/my-image.png %}`. Шаблон не содержит этого плагина, но его можно скопировать в директорию `_plugins/`и использовать.
## Развёртывание в GitHub Pages
После того, как сайт получится развернуть локально, можно перейти к развёртыванию в интернете. Один из самых простых способов - это использовать [[Github Actions]] и [[Github Pages]].

Алгоритм конфигурации Actions и Pages:
- Создать [[GitHub#Репозиторий|репозиторий]] и загрузить туда шаблон или склонировать репозиторий с шаблоном
- Удалить `.github/workflows/jekyll.yml`файл. Шаблон содержит старый способ развёртывания. Далее будет описана его замена.
- Открыть Github, перейти в репозиторий и открыть Settings > Pages
- Выбрать `Source: Gtihub Actions`. После этого Github предложит сконфигурировать [[Github Actions#Workflow|workflow]]. Выбор Jekyll workflow создаст файл `.github/workflows/jekyll-gh-pages.yaml`, который уже будет содержать все необходимые шаги для развёртывания
- Добавление workflow запустит Action, который развернёт. сайт на Github Action.

## Добавление собственного домена для Github Pages
Алгоритм изменений домена для [[GoDaddy]]:
- Завести домен на GoDaddy
- Сконфигурировать в настройках домена интеграцию с GitHub (более подробно тут: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain)
- В корне репозитория изменить файл CNAME - прописать туда свой DNS
- В настройках репозитория в `Settings > Pages > Custom domain` указать адрес (www.name.com и name.com - это разные адреса, нужно ознакомиться с инструкцией)