---
title: Disposing and CA2000
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
## Dispose objects before losing scope (CA2000)

Существует несколько стандартных [[Roslyn analyzers|Roslyn анализаторов]], которые находят проблемы с использованием [[Instance disposing|IDisposable]]:
- CA2000
- CA2213

Документация к анализатору CA2000 даёт такое описание:
> If a disposable object is not explicitly disposed before all references to it are out of scope, the object will be disposed at some indeterminate time when the garbage collector runs the finalizer of the object. Because an exceptional event might occur that will prevent the finalizer of the object from running, the object should be explicitly disposed instead.

Этот анализатор находит места, где внутри метода создался экземпляр класса с IDisposable реализацией, но не был вызван метод Dispose. Основная проблема данного анализатора в том, что он не умеет находить owner transferring и генерирует из-за этого false positive. Подробно это описано в ишуях https://github.com/dotnet/roslyn-analyzers/issues/1617 и https://github.com/dotnet/runtime/issues/29631.

Но такое правило бы генерировало слишком много false positive. Поэтому для поддержки фабричных методов была сделано ещё одно допущение: если экземпляр создаётся и отдаётся возвращаемым значением, то это также передаёт ownership вызывающему коду:

```csharp
private FileStream Create()
{
  return new FileStream("path", FileMode.Create); // Ok
}

private void Use()
{
  var fileStream = Create(); // Warning
}
```

Со стороны вызывающего кода также работает такая семантика:
```csharp
public static Process GetProcess()
{
  return Process.GetCurrentProcess();
}

public static Process CreateProcess()
{
  return Process.GetCurrentProcess();
}

Process process1 = GetProcess(); // Ok
Process process2 = CreateProcess(); // Warning
```

## CA2000: Избавление от false positive

При этом есть несколько неочевидных и не задокументированных способов повлиять на поведение анализатора, чтобы снизить количество false positive.

Первый способ - это ["Special cases"](https://github.com/dotnet/docs/blob/main/docs/fundamentals/code-analysis/quality-rules/ca2000.md#special-cases) для Stream, StreamReader и подобных типов. Особенность заключается в том, что для этих типов в коде анализатора прописано, что они берут на себя ownership, и значит любой инстанс, который передаётся в конструктор StreamReader будет освобождаться от необходимость быть задиспоуженым явно.
```csharp
public void M1()
{
  var fileStream = new FileStream("C:/myfile.txt", FileMode.Create); // Warning
}

public void M2()
{
  var fileStream = new FileStream("C:/myfile.txt", FileMode.Create); // Ok
  var reader = new StreamReader(fileSteam);
}
```

Второй способ - это конфигурации анализатора с помощь `dispose_ownership_transfer_at_constructor` и `dispose_ownership_transfer_at_method_call`.
При выставлении `at_constructor` [[Roslyn Data flow analys]] будет считать, что для всех передаваемых IDisposable экземпляров также передаётся и ownership .
```csharp
public void Method()
{
  // Default: Warning
  // with dispose_ownership_transfer_at_constructor: Ok
  return new MyType(new FileStream("path", FileMode.Create));
}
```

Но данные опции ослабляют анализатор и открывают возможность для false negative. Без опции вызов Dispose требуется от Method и не требуется от конструктора. С включённой опцией вызов Dispose не требуется ни от Method, ни от конструктора.

Описание этих опций можно найти на [GitHub](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/Analyzer%20Configuration.md#configure-dispose-ownership-transfer-for-disposable-objects-passed-as-arguments-to-method-calls).

