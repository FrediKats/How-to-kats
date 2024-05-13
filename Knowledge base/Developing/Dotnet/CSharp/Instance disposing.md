---
title: Instance disposing
---

IDisposing - это интерфейс с единственным методом Dispose. Концептуально этот метод нужен для того, чтобы выполнить код после того, как объект перестанет использоваться. Пример ситуации, когда это необходимо - это работа с внешними ресурсами. При создании некоторых объектов для работы с файлами, приложение захватывает файловый дескриптор у операционной системы и не даёт другим приложениям работать с файлом, пока приложение не даст сигнал, что дескриптор больше не удерживается.

```csharp
var file = new File();
// Some operation with file
file.Dispose();
```

Для упрощения работы с IDisposable в C# есть конструкция `using`:
```csharp
using (var file = new File())
{
  // Some operation with file
} // .Dispose will call after exit from statement context
```
## Ownership transferring 
Простые сценарии работы с IDisposable - это создание, использование и вызов Dispose в рамках одного метода. Но экземпляр может передаваться в другие методы. Microsoft рекомендуют придерживаться алгоритм: "если создаётся диспосабельный экземпляр, то нужно явно вызывать метод Dispose":
```csharp
public void F1()
{
	using (var file = new File())
	  Do(file);
}
```

Но есть сценарий, когда такой подход не сработает - создание экземпляра. Допустим, что нам нужно создать экземпляр класса MyClass. Одним из аргументов конструктора данного класса является тип, который реализует IDisposable. И допустим, что код создание экзмпляра нужно вынести в фабричный метод. Получим такой код:
```csharp
public MyClass Create()
{
  var file = new File();
  return new MyClass(file);
}
```

В данном случае происходит ownership transferring, экземпляр File передаётся в MyClass и ожидается, что он будет управлять жизненным циклом и вызовет метод Dispose в своём методе Dispose.
Ещё одним примером является API типа SteamReader:
```csharp
using (var reader = new StreamReader(new FileStream("path", FileMode.Create)))
```
По умолчанию считается, что StreamReader в такой ситуации захватывает управление экземпляром FileStream.

## Повторные вызовы Dispose
В рекомендациях от Microsoft указано, что вызов Dispose метода должен быть идемпотентным. Повторные вызовы не должны бросать ошибки или приводить к другим сайд-эффектам. Но к сожалению не все разработчики библиотек следуют этому и даже стандартные типы не всегда себя так ведут. Так например, повторные вызовы Dispose у TcpClient генерируют ошибки о том, что объект уже был задиспожен.

Более подробно описано тут: https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose.
## Вызовы Dispose в конструкторе
Dispose – это метод экземпляра и вызывается на нём. Из этого следует очень неприятный вывод: "нельзя вызвать Dispose, если нет объекта". Представим ситуацию, что в конструкторе создаётся два экземпляра класса Stream и сохраняются в поля:
```csharp
public class MyType : IDisposable
{
  private readonly Stream _stream1;
  private readonly Stream _stream2;
  public MyType()
  {
    _stream1 = new FileStream();
    _stream2 = new FileStream();
  }
  public void Dispose()
  {
    _stream1.Dispose();
    _stream2.Dispose();
  }
}

using (var value = new MyType())
  // Some logic
```

Такая реализация не будет корректно обрабатывать ситуацию, когда создание второго потока завершится с ошибкой. Это приведёт к тому, что экземпляр типа MyType не будет создан, а значит у него не может быть вызван Dispose. Созданный экземпляр \_stream1 останется без owner'а. Варианты решения проблемы:
- Обернуть код создания экземпляров FileStream в try/catch и освобождать первый, если второй не создался
- Обернут код в конструкторе в try/catch и вызывать Dispose

## Ссылки
- [Cleaning up unmanaged resources | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged)