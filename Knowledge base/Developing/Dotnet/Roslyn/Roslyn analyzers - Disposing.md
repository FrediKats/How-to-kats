---
title: Roslyn Analyzers - Disposing
---

Существует несколько стандартных [[Roslyn analyzers|Roslyn анализаторов]], которые находят проблемы с использованием [[Instance disposing|IDisposable]]:
- CA2000
- CA2213
## Dispose objects before losing scope (CA2000)
  
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

