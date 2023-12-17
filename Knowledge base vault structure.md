[[Obsidian#Obsidian vault|Obsidian vault]] имеет древовидную структуру, как файловая система. Это не самый удобный способ представления графа знаний, поэтому для создания базы знаний нужно описать набор правил организации страниц.

Есть несколько стандартных директорий, существование которых стоит учитывать:
- Templates для [[Obsidian Templates|шаблонов]]
- Attachments для хранения добавленных изображений (добавляются автоматически при ctrl+v)

Предполагаемая структура:
```
- Attachments
- Inbox
- Knowledge base
	- Code
		- Code Design
		- Dotnet
		- Multithreading
		- ...
	- Computer science
		- Algorithm
		- Benchmarking
		- Virtualization
		- ...
	- Database
	- Math
	- Project management
	- Tools
- Materials
	- Articles
	- Books
	- Videos
- People
- Personal content
	- Articles
	- How to
	- Meetups
```

Notes:
- Директория People содержит страницы с краткой биографией людей. Это позволит в страницах про доклады или книги ссылаться на них.