# Panic 异常与 defer
{docsify-updated}

> https://go.dev/blog/defer-panic-and-recover

## Panic

Go的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起panic异常。

一般而言，当panic异常发生时，程序会中断运行，并立即执行在该goroutine（可以先理解成线程，在第8章会详细介绍）中被 `defer` 的函数（defer 机制）。随后，**程序崩溃**并输出日志信息。日志信息包括panic value和函数调用的堆栈跟踪信息。panic value通常是某种错误信息。对于每个goroutine，日志信息中都会有与之相对的，发生panic时的函数调用堆栈跟踪信息。通常，我们不需要再次运行程序去定位问题，日志信息已经提供了足够的诊断依据。因此，在我们填写问题报告时，一般会将panic异常和日志信息一并记录。

**注意 panic 异常会导致程序崩溃。**

不是所有的panic异常都来自运行时，直接调用内置的panic函数也会引发panic异常；panic函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用panic。比如，当程序到达了某条逻辑上不可能到达的路径：
```
switch s := suit(drawCard()); s {
case "Spades":                                // ...
case "Hearts":                                // ...
case "Diamonds":                              // ...
case "Clubs":                                 // ...
default:
    panic(fmt.Sprintf("invalid suit %q", s)) // Joker?
}
```

虽然Go的panic机制类似于其他语言的异常，但panic的适用场景有一些不同。**由于panic会引起程序的崩溃，因此panic一般用于严重错误，如程序内部的逻辑不一致**。勤奋的程序员认为任何崩溃都表明代码中存在漏洞，所以对于大部分漏洞，我们应该使用Go提供的错误机制，而不是panic，尽量避免程序的崩溃。在健壮的程序中，任何可以预料到的错误，如不正确的输入、错误的配置或是失败的I/O操作都应该被优雅的处理，最好的处理方式，就是使用Go的**错误机制**。

## defer
`defer` 语句将函数调用推入一个栈。 保存的调用列表会在声明 `defer` 语句的函数返回后执行。 这种 `defer` 调用通常用于简化执行各种清理操作的函数。 `defer` 有以下几个特点：
1. `defer` 函数的参数在 `defer` 语句执行时求值（在声明`defer`语句的那一刻，函数参数的值在那个时刻确定）。
   ```
   func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
   }
   ```
   上面的代码打印 0
2. `defer` 函数调用在声明它的函数执行完成后按后进先出的顺序执行。
   ```
   func b() {
    for i := 0; i < 4; i++ {
        defer fmt.Print(i)
    }
   }
   ```
   上述代码打印 3210
3. `defer` 函数可以读取及操作函数的命名返回值。
   ```
   func c() (i int) {
    defer func() { i++ }()
    return 1
   }
   ```
   函数返回 2


## defer 与 recover
panic 是一个内置函数，用于停止普通的控制流并开始 panic。 当函数 F 调用 panic 时，F 的执行会停止，F 中的任何 `defer` 函数都会正常执行，然后 F 返回给调用者。 对于调用者来说，F 的行为就像调用 panic。 这个过程会在堆栈上继续进行，直到当前 goroutine 中的所有函数都返回为止，这时程序就会崩溃。 恐慌可以通过直接调用 panic 引发。 运行时错误也可能导致恐慌，例如越界数组访问。

Recover 是一个内置函数，用于恢复对慌乱的 goroutine 的控制。 **Recover 仅在递延函数内部有用**。 在正常执行过程中，调用 recover 将返回 nil，不会产生其他影响。 如果当前的 goroutine 正在 panic，调用 recover 将捕获 panic 的值并恢复正常执行。

```
ackage main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```
输出：
```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```