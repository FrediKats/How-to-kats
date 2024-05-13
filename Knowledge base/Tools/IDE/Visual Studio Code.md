---
title: Visual Studio Code
---
Visual Studio Code - это [[IDE]] от Microsoft.
## Adding to context menu
Стандартная установка VS Code добавляет в контекстное меню команду открытия директории в VS Code. При использовании [[winget]] VS Code устанавливается без этого. Чтобы добавить VS Code в контекстное меню после установки можно воспользоваться командой:

```
Windows Registry Editor Version 5.00
; This will handle right clicking on a file

[HKEY_CLASSES_ROOT\*\shell\Open with VS Code]
@="Edit with VS Code"
"Icon"="C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe,0"

[HKEY_CLASSES_ROOT\*\shell\Open with VS Code\command]
@="\"C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe\" \"%1\""

; This will handle right clicking on a folder and open that folder
; as a new project

[HKEY_CLASSES_ROOT\Directory\shell\vscode]
@="Open Folder as VS Code Project"
"Icon"="\"C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe\",0"

[HKEY_CLASSES_ROOT\Directory\shell\vscode\command]
@="\"C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe\" \"%1\""

; This handles the case of right clicking inside of a folder
; to open that folder as a new project

[HKEY_CLASSES_ROOT\Directory\Background\shell\vscode]
@="Open Folder as VS Code Project"
"Icon"="\"C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe\",0"

[HKEY_CLASSES_ROOT\Directory\Background\shell\vscode\command]
@="\"C:\\Users\\User\\AppData\\Local\\Programs\\Microsoft VS Code\\Code.exe\" \"%V\""
```