---
title: .editorconfig
---

.editorconfig - это специальный файл, который используется для описания кодстайла и подобных настроек кода. Это универсальный инструмент, который не привязан к конкретному языку, а поддерживается многими разными языками. Большинство современных IDE также умеют работать с этим файлом.

.editorconfig имеет структуру ini файла:
```ini
# top-most EditorConfig file
root = true

[*]
insert_final_newline = false
charset = utf-8
indent_style = space
indent_size = 4
```

Сайт проекта - https://editorconfig.org/