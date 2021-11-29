---
title: 了解go语言
date: 2021-08-09 17:47:11
tags:
---



# 官方地址

> https://golang.org/

Go是一个开放源代码编程语言，它使得 构建 **简单**，**可靠**和**高效**的软件更容易。

# 安装go

主页上这个download go， 即可以下载go. 并且下面说 二进行发行，对linux, mac, windows 甚至更多的平台可用。

![image-20210809174946785](http://myapp.img.mykernel.cn/image-20210809174946785.png)

点击后即进入安装界面，选择windows包。

下载适合你系统的二进制包后，遵循以下[安装指令](https://golang.org/doc/install)

进入安装指令后，会介绍3个平台的安装方法，以下 为windows安装:

1. Open the MSI file you downloaded and follow the prompts to install Go.

   By default, the installer will install Go to `Program Files` or `Program Files (x86)`. You can change the location as needed. After installing, you will need to close and reopen any open command prompts so that changes to the environment made by the installer are reflected at the command prompt.

2. Verify that you've installed Go.

   1. In **Windows**, click the **Start** menu.

   2. In the menu's search box, type `cmd`, then press the **Enter** key.

   3. In the Command Prompt window that appears, type the following command:

      ```
      $ go version
      ```

   4. Confirm that the command prints the installed version of Go.

# 写go代码

You're set up! Visit the [Getting Started tutorial](https://golang.org/doc/tutorial/getting-started.html) to write some simple Go code. It takes about 10 minutes to complete.

## 简单的hello world

```bash
cd %HOMEPATH%
mkdir hello
cd hello
go mod init github.com/hello # 当前目录作为一个模块。此为模块的url可访问路径。
```

> 生成的`go.mod`
>
> ```go
> module github.com/hello
> 
> go 1.16
> ```

编辑`hello.go`

```go
package main  //当前目录
import "fmt"

func main() {
    fmt.Println("hello world")
}
```

> 在vscode上安装`Code Runner`, 直接点右上角执行. 或者在windows的cmd` go run hello.go`或`go run .`

## 加载外部的模块

进入[pkg.go.dev](http://pkg.go.dev) ，搜索`quote`包，进入rsc.io/quote.

在document，index下，列出了你可以在代码中调用的函数列表。

```go
func Glass() string // 返回string
func Go() string
func Hello() string
func Opt() string
```

在页面顶部显示 discover packages > `rsc.io/quote` 模块。quote是包

代码中引用包`hello.go`

```go
package main // 当前目录的模块
import "fmt" // go标准包
import "rsc.io/quote" // 导入外部模块

func main() {
	fmt.Println("Hello, world!")
	fmt.Println(quote.Go())
}
```

添加quote模块和go.sum文件作为依赖. `go get或go mod tidy` 均可.

```bash
# 由于网络不好，使用代理：
powershell
$env:GOPROXY = "https://goproxy.io,direct"

# 获取包
C:\Users\rwx\hello>go mod tidy # it located and downloaded the rsc.io/quote module  that contains the package you imported.
go: finding module for package rsc.io/quote
```

> 现在可以发现`go.mod`中的变化
>
> ```go
> module github.com/hello
> 
> go 1.16
> 
> require rsc.io/quote v1.5.2
> ```



再次运行

```powershell
PS C:\Users\rwx\hello> go run .
Hello, world!
Don't communicate by sharing memory, share memory by communicating.
```



## 创建一个模块

这个教程次序包含简洁的7个话题，每一次阐述语言的一部分

1. 创建一个模块 -- 写一个简洁的功能模块，你可以在其他module中调用。
2. 从其他module中调用你的代码 -- 导入并使用你的新module
3. 返回并处理错误  -- 添加简单的错误处理。
4. 返回一个随机的问题 -- 使用切片处理数据(go的动态大小的arrays)
5. 返回对多人问候 -- 在map中存储Key/value
6. [Add a test](https://golang.org/doc/tutorial/add-a-test.html) -- 使用go内建的test特性来测试你的代码。
7. [Compile and install the application](https://golang.org/doc/tutorial/compile-install.html) -- Compile and install your code locally.

### 创建一个其他人可以使用的模块

在一个模块中，可以集合一个或多个离散和有用的功能集合的包。比如，你可以创建一个带有做金融分析的包的模块，当写金融应用的其他人就可以使用你的工作成果。 For more about developing modules, see[Developing and publishing modules](https://golang.org/doc/modules/developing).

Go代码组织成packages, 多个packages组装进多个modules。你的模块中会指明，运行你代码所需的依赖。包括go的版本和requires一堆其他modules.

你在模块中添加和导入功能之后，你就可以发布模块的新功能了。开发者在把代码推到生产环境使用之前会，写代码。在你的模块中调用的函数可以导入更新的packages和使用新版本test.

准备模块目录

```bash
$ mkdir create_module/greetings
$ cd create_module/greetings
```

```bash
# 初始化模块，并指定模块的可下载路径
$ go mod init github.com/greetings
go: creating new go.mod: module github.com/greetings
```

在`greetings.go`文件中编写greetings包

```go
// Create a module -- Write a small module with functions you can call from another module.

package greetings

import "fmt"
//      go函数大写起始         接受string类型，返回string类型
// 此package, 可以被不在此包中调用
func Hello(name string) string {
	// 申明变量 message.     
	// := declaring and initializing a variable. 右边的值决定变量的类型。
	// 其实等同于 如下2行代码
	// var message string
	// message = fmt.Sprintf("Hi, %v. Welcome", name)

	// Sprintf, 参数1: 格式化字符，Sprintf替换name的值%v
	message := fmt.Sprintf("Hi, %v. Welcome", name)
	return message
}
```

在此节中，创建了一个`greetings` module. 下一节你将写可以执行为一个程序的代码，并调用`greetings` module的`Hello`函数。

### 从其他module中调用你的代码

准备模块目录

```bash
$ cd ../
$ mkdir hello
```

> 目录结构
>
> ```bash
> <home>/
>  |-- greetings/
>  |-- hello/
> ```

```bash
# 初始化模块，并指定此模块可下载git地址。
$ cd hello/
$ go mod init github.com/hello
go: creating new go.mod: module github.com/hello
```

在`hello.go`文件中写一个可以执行为程序的包`main`

```go
// Call your code from another module -- Import and use your new module.

package main  // In Go, code executed as an application must be in a main package.

import (
    "fmt" // handling input and output text (such as printing text to the console).

    "github.com/greetings" // the package contained in the module you created earlier Hello function
) // Import two packages

func main() {
    // Get a greeting message and print it.
    message := greetings.Hello("Gladys") // Get a greeting by calling the greetings package’s Hello function.
    fmt.Println(message)
}
```

> 如果在生产中使用，你应用发布greetings module到初始化指定的repository，然后go tool就可以找到并下载模块。
>
> 但是现在由于并没有发布greetings module到github.com/greetings模块，就需要适应github.com/greetings模块，hello模块只需要在本地文件系统中查找greetings模块。
>
> 通过go mod edit命令，让Go tool 知道从本地查找模块
>
> ```bash
> rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
> $ go mod edit -replace github.com/greetings=../greetings
> ```
>
> > 现在可以观察到go.mod文件, 替换url为相对路径
> >
> > ```go
> > module github.com/hello
> > 
> > go 1.16
> > 
> > replace github.com/greetings => ../greetings
> > ```
>
> 现在在获取依赖
>
> ```bash
> $ go mod tidy
> go: found github.com/greetings in github.com/greetings v0.0.0-00010101000000-000
> 000000000
> ```
>
> > 现在观察`go.mod`文件, 通过require指定hello模块需要greetings模块。
> >
> > 跟在模块路径后面的版本号，使用`semantic number`语义化版本号，模块本身并没有提供。
> >
> > ```go
> > module github.com/hello
> > 
> > go 1.16
> > 
> > replace github.com/greetings => ../greetings
> > 
> > require github.com/greetings v0.0.0-00010101000000-000000000000
> > ```
> >
> > **引用一个模块，通常会省略replace, 只会 require有一个带版本的模块**
> >
> > For more on version numbers, see [Module version numbering](https://golang.org/doc/modules/version-numbers).
> >
> > 语义化版本：v major.minor.patch-beat/alpha.x
> >
> > - v0.x.x 开发过程中，并且**不保证稳定和兼容**
> >
> > - major: 不保证向下**兼容**api，v1及以上版本表明模块是稳定的。v0是不稳定和不兼容，开发者从v0升级到v1时，有负责适应这种不兼容带来的影响。 
> >
> > 更新版本号高于v1的主版本将有一个新的模块路径。模块路径后面会追加一个版本号。例如` module example.com/mymodule/v2 v2.0.0`
> >
> > ​	主版本更新，使得新模块有较之前版本有一个独立的历史。 If you're developing modules to publish for others, see "Publishing breaking API changes" in [Module release and versioning workflow](https://golang.org/doc/modules/release-workflow).
> >
> > - minor: 保证向下**兼容**api。只是enhancements, This might include changes to a module’s own dependencies or the addition of new functions, methods, struct fields, or types.
> >
> > - patch: 不影响api，仅仅当minor中bug修复等微小变化才更新此数字。开发者不修改代码就可以安全升级此版本。
> >
> > - beta/alpha: 不保证**稳定**

在hello模块目录中，运行

```bash
$ go run .
Hi, Gladys. Welcome
```



Congrats! 你已经写了2个功能模块。下一个话题，你将写一些错误处理。

### 添加简单的处理错误

处理错误是固定代码的必要的特性。这节，你将从greetings module中返回一个错误，然后调用中处理它。

`greetings/greetings.go` 添加以下高亮部分

```diff
// Create a module -- Write a small module with functions you can call from another module.

package greetings

import (
+        "errors" //  Go standard library errors package 
        "fmt")
//      go函数大写起始         接受string类型，返回string类型
// 此package, 可以被不在此包中调用
+ func Hello(name string) (string,error) {
+        // If no name was given, return an error with a message.
+        if name == "" {
+                return "", errors.New("empty name")
+        }

        // 申明变量 message.
        // := declaring and initializing a variable. 右边的值决定变量的类型。
        // 其实等同于 如下2行代码
        // var message string
        // message = fmt.Sprintf("Hi, %v. Welcome", name)

        // Sprintf, 参数1: 格式化字符，Sprintf替换name的值%v
        message := fmt.Sprintf("Hi, %v. Welcome", name)
+        return message, nil
}
```

> so that it returns two values: a `string` and an `error`. Your caller will check the second value to see if an error occurred. (Any Go function can return multiple values. For more, see[Effective Go](https://golang.org/doc/effective_go.html#multiple-returns).)
>
> The `errors.New` function returns an `error` with your message inside.
>
> Add `nil` (meaning no error) as a second value in the successful return. That way, the caller can see that the function succeeded.

`hello/hello.go`, 处理Hello函数返回的错误。

```diff
// Call your code from another module -- Import and use your new module.

package main  // In Go, code executed as an application must be in a main package.

import (
    "fmt" // handling input and output text (such as printing text to the console).
+	"log"  // in the standard library's log package，打印日志消息："greetings: "起始，不带时间戳和源文件信息
    
    "github.com/greetings" // the package contained in the module you created earlier Hello function
) // Import two packages

func main() {
+    // Set properties of the predefined Logger, including
+    // the log entry prefix and a flag to disable printing
+    // the time, source file, and line number.
+    log.SetPrefix("greetings: ")
+    log.SetFlags(0)
    
+    // Request a greeting message. 
+    message, err := greetings.Hello("")
+    // If you get an error, you use the log package's Fatal function to print the error and stop the program.
+    if err != nil {
+        log.Fatal(err)
+    }

+    // If no error was returned, print the returned message
+    // to the console.
+    fmt.Println(message)
}
```

运行命令

```bash
$ go run .
greetings: empty name
exit status 1
```

### 返回一个随机的问候

这节将修改代码，仅返回多个预定义的问题消息其中的一个。

使用`go slice`, 类似于`array`, 一旦添加或删除items时将会动态改变其大小。`slice`是go最有用类型之一。 

你将添加一个小的 包含3个问候消息的slice，然后你的代码返回随机的一个。For more on slices, see [Go slices](https://blog.golang.org/slices-intro) in the Go blog.

`greetings/greetings.go`

```diff
// Create a module -- Write a small module with functions you can call from another module.

package greetings

import (
        "errors"
        "fmt"
+		"math/rand" // 生成随机数，来从slice中选择一个
    	"time"
)
//      go函数大写起始         接受string类型，返回string类型
// 此package, 可以被不在此包中调用
func Hello(name string) (string, error) {
        // If no name was given, return an error with a message.
        if name == "" {
                return "", errors.New("empty name") // returns two values: a string and an error.
        }


        // 申明变量 message.
        // := declaring and initializing a variable. 右边的值决定变量的类型。
        // 其实等同于 如下2行代码
        // var message string
        // message = fmt.Sprintf("Hi, %v. Welcome", name)

        // Sprintf, 参数1: 格式化字符，Sprintf替换name的值%v
+    // Create a message using a random format.
+    message := fmt.Sprintf(randomFormat(), name)
        return message, nil
}


+ // init sets initial values for variables used in the function.
+ func init() {
+     rand.Seed(time.Now().UnixNano())
+ } // Go executes init functions automatically at program startup， after global variables have been initialized。

+ // randomFormat returns one of a set of greeting messages. The returned
+ // message is selected at random.
+ func randomFormat() string {
+     // A slice of message formats.
+     formats := []string{
+         "Hi, %v. Welcome!",
+         "Great to see you, %v!",
+         "Hail, %v! Well met!",
+     }

+     // Return a randomly selected message format by specifying
+     // a random index for the slice of formats.
+     return formats[rand.Intn(len(formats))]
+ }
```

> randomFormat函数返回一个随机的被选择的格式。注意 `randomFormat`以**小写**字母开始，使得它仅在自己的包中能访问（**不会被导出**）
>
> 在`randomFormat`中申明了`formats` slice. 当申明一个slice时，你在`brackets`方括号中省略其大小时。像这样`[]string`，这将告知Go, 以slice为基础的array的大小可以动态的修改。
>
> init函数，在全局变量初始化后，会自动执行。For more about `init`functions, see [Effective Go](https://golang.org/doc/effective_go.html#init).

`hello/hello.go` 接受一个正常的名称

```go
message, err := greetings.Hello("Gladys")
```

运行

```bash
rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
$ go run .
Great to see you, Gladys

rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
$ go run .
Great to see you, Gladys

rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
$ go run .
Hail, Gladys! Well met!

rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
$ go run .
Great to see you, Gladys
```

> 运行多次，注意消息的变化。

接下来，你将使用一个slice问候多个人

### 返回对多个人的问候

你将处理多值输入，传递一组名称给函数，它返回对他们每一个问候

有一个问题，将修改Hello函数参数为一组名称。将修改函数的签名。如果你的模块已经发布到`github.com/hello`，并且 已经有人在使用你的Hello代码，这个修改将使他们的程序崩溃。

这种情况下，最好写一个拥有新名称的新的函数，新函数可以接受多个值，这次保留老的函数，为了向后兼容。

`greetings/greetings.go`, 修改代码

```diff
// Create a module -- Write a small module with functions you can call from another module.

package greetings

import (
        "errors"
        "fmt"
        "math/rand"
        "time"
)
//      go函数大写起始         接受string类型，返回string类型
// 此package, 可以被不在此包中调用
func Hello(name string) (string, error) {
        // If no name was given, return an error with a message.
        if name == "" {
                return "", errors.New("empty name") // returns two values: a string and an error.
        }


        // 申明变量 message.
        // := declaring and initializing a variable. 右边的值决定变量的类型。
        // 其实等同于 如下2行代码
        // var message string
        // message = fmt.Sprintf("Hi, %v. Welcome", name)

        // Sprintf, 参数1: 格式化字符，Sprintf替换name的值%v
        message := fmt.Sprintf(randomFormat(), name)
        return message, nil
}




func init() {
        rand.Seed(time.Now().UnixNano())
}


func randomFormat() string {
        formats := []string {
                "Hi, %v, Welcome!",
                "Great to see you, %v",
                "Hail, %v! Well met!",
        }

        return formats[rand.Intn(len(formats))]
}

+// Hellos returns a map that associates each of the named people
+// with a greeting message.
+func Hellos(names []string) (map[string]string, error) {
+    // A map to associate names with messages.
+    messages := make(map[string]string)
+    // Loop through the received slice of names, calling
+    // the Hello function to get a message for each name.
+    for _, name := range names {
+        message, err := Hello(name)
+        if err != nil {
+            return nil, err
+        }
+        // In the map, associate the retrieved message with
+        // the name.
+        messages[name] = message
+    }
+    return messages, nil
+}
```

> Hellos函数的参数是`names []string` 不是单一的名称，它返回之一的类型也修改成map。
>
> 这个函数会调用Hello函数
>
> 创建一个messages的map, 它关联了每一个接受的name和对应的生成的消息。在go中初始化一个map, 使用`make(map[key-type]value-type)`, For more about maps, see [Go maps in action](https://blog.golang.org/maps) on the Go blog.
>
> In this `for` loop, `range` returns two values: the index of the current item in the loop and a copy of the item's value.
>
> You don't need the index, so you use the Go blank identifier (an underscore) to ignore it. For more, see [The blank identifier](https://golang.org/doc/effective_go.html#blank) in Effective Go.

`hello/hello.go`

```diff
// Call your code from another module -- Import and use your new module.

package main // In Go, code executed as an application must be in a main package.

import (
        "fmt" // handling input and output text (such as printing text to the console).
        "log"

        "github.com/greetings" // the package contained in the module you created earlier Hello function
) // Import two packages

func main() {
        log.SetPrefix("greetings: ")
        log.SetFlags(0)
        // Get a greeting message and print it.
+        // A slice of names.
+        names := []string{"Gladys", "Samantha", "Darrin"}
+        messages, err := greetings.Hellos(names) // Get a greeting by calling the greetings package’s Hello function.
        if err != nil {
                log.Fatal(err)
        }
+        fmt.Println(messages)
}

```

> - Create a `names` variable as a slice type holding three names.
> - Pass the `names` variable as the argument to the `Hellos` function.

运行命令

```bash
$ go run hello.go
map[Darrin:Hail, Darrin! Well met! Gladys:Hail, Gladys! Well met! Samantha:Hi, S
amantha, Welcome!]
```

本主题介绍了`map`表示name/value对。也介绍了在模块中新功能或修改功能时，通过实现新函数向后兼容。For more about backward compatibility, see [Keeping your modules compatible](https://blog.golang.org/module-compatibility).

接下来，你将使用内建的go特性来为你的代码建立单元测试

### 添加测试

在开发期间测试你的代码，可以暴露你的代码的bug, 找到后以自己的方式修正。这篇主题将为Hello函数添加测试。

go内建支持单元测试。使得你写go很容易测试，go的测试package和go test命令，可以很快写测试和执行测试。

在greetings目录，创建一个`greetings_test.go`

```bash
$ cd ../greetings/
$ touch greetings_test.go
```

> _test.go结束的文件名，告诉go命令，这个文件包含了测试函数

`greetings_test.go`

```go
package greetings

import (
    "testing"
    "regexp"
)

// TestHelloName calls greetings.Hello with a name, checking
// for a valid return value.
func TestHelloName(t *testing.T) {
    name := "Gladys"
    want := regexp.MustCompile(`\b`+name+`\b`)
    msg, err := Hello("Gladys")
    if !want.MatchString(msg) || err != nil {
        t.Fatalf(`Hello("Gladys") = %q, %v, want match for %#q, nil`, msg, err, want)
    }
}

// TestHelloEmpty calls greetings.Hello with an empty string,
// checking for an error.
func TestHelloEmpty(t *testing.T) {
    msg, err := Hello("")
    if msg != "" || err == nil {
        t.Fatalf(`Hello("") = %q, %v, want "", error`, msg, err)
    }
}
```

> 测试函数命名: `TestName`, 命名也说明特定的测试。
>
> test测试接受一个pointer，使用testing package中的 testing.T作为参数. 使用参数的方法来报告 和记录你的测试。
>
> 实现2个测试:
>
> - `TestHelloName` calls the `Hello` function, 传递一个name变量，函数应该返回一个可用的响应。如果返回错误或不期望的消息，使用t参数的`Fatalf`方法打印消息到控制台，并结束执行。
> - `TestHelloEmpty` calls the `Hello` function with an empty string. 这个测试用来确定你的错误处理是工作的。如果调用返回非none或没有错误发生，就使用t参数的`Fatalf`方法打印消息到console并退出执行。

在greetings目录中，执行`go test`命令来执行测试，这个命令将会执行_test.go结尾的文件中的Test起始的函数。可以加`-v` 标识获取测试详细和结果的详细。

测试通过

```bash
$ go test
PASS
ok      github.com/greetings    0.242s

rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/greetings
$ go test -v
=== RUN   TestHelloName
--- PASS: TestHelloName (0.00s)
=== RUN   TestHelloEmpty
--- PASS: TestHelloEmpty (0.00s)
PASS
ok      github.com/greetings    0.280s

```

破坏greetings.Hello函数，查看失败的测试

`greetings/greetings.go`, 不给Hello函数的Sprintf传递name

```diff
//      go函数大写起始         接受string类型，返回string类型
// 此package, 可以被不在此包中调用
func Hello(name string) (string, error) {
        // If no name was given, return an error with a message.
        if name == "" {
                return "", errors.New("empty name") // returns two values: a string and an error.
        }


        // 申明变量 message.
        // := declaring and initializing a variable. 右边的值决定变量的类型。
        // 其实等同于 如下2行代码
        // var message string
        // message = fmt.Sprintf("Hi, %v. Welcome", name)

        // Sprintf, 参数1: 格式化字符，Sprintf替换name的值%v
        // message := fmt.Sprintf(randomFormat(), name)
    	message := fmt.Sprintf(randomFormat())
    
        return message, nil
}
```

执行测试的结果
The `TestHelloName` test should fail

`TestHelloEmpty` still passes.

```bash
rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/greetings
$ go test
--- FAIL: TestHelloName (0.00s)
    greetings_test.go:15: Hello("Gladys") = "Great to see you, %!v(MISSING)", <n
il>, want match for `\bGladys\b`, nil
FAIL
exit status 1
FAIL    github.com/greetings    0.248s

```

先恢复hello函数

### 在本地编译安装你的代码

go run 是编译和运行的快捷方式

go build  连同依赖一起编译包，但不安装编译包

go install 编译和安装包



hello目录中运行`go build`, 编译代码为可执行

```bash
$ go build

rwx@DESKTOP-HHT6P69 MINGW64 ~/hello/create_module/hello
$ ls
go.mod  hello.exe*  hello.go
```

> 在不同的操作系统上，编译的结果不同。

运行

```bash
$ ./hello.exe
map[Darrin:Great to see you, Darrin Gladys:Hi, Gladys, Welcome! Samantha:Hail, Samantha! Well met!]

```

查看go的安装路径

```bash
$ go list -f '{{.Target}}'
C:\Users\rwx\go\bin\hello.exe
```

添加此路径到系统环境

```bash
# windows
set PATH=%PATH%;C:\path\to\your\install\directory

# linux
export PATH=$PATH:/path/to/your/install/directory
```

或者直接通过`go install`安装到你的安装目录下

```bash
go env -w GOBIN=/path/to/your/bin
or
go env -w GOBIN=C:\path\to\your\bin
```

现在在安装程序到上面指定的位置

```bash
go install
```

现在只需要简洁使用名字来执行程序。

```bash
$ hello
map[Darrin:Hail, Darrin! Well met! Gladys:Great to see you, Gladys! Samantha:Hail, Samantha! Well met!]
```

## 其他教程

https://golang.org/doc/tutorial/

> 使用 page up/ page down前后翻

## 访问关系型数据库

使用go和其标准库`database/sql`访问关系型数据库的基础, 也可以使用上面[加载外部的模块](#加载外部的模块)来搜索互联网上的访问关系型数据库的模块。

使用这个包的详细教程[Accessing databases](https://golang.org/doc/database/index).

### 依赖

- **一个安装的 [MySQL](https://dev.mysql.com/doc/mysql-installation-excerpt/5.7/en/) relational database management system (DBMS).**
- **一个安装的 Go.**  安装向导 [Installing Go](https://golang.org/doc/install).
- 编辑代码的工具。你使用的任何文本编辑工具，推荐: notepad, sublime text, vim, nano
- 一个命令行终端。go在linux/mac/windows上的任何终端都工作的很好

### 为你的代码创建一个目录

1. 为调用data-access的代码创建目录

```bash
$ install -dv data-access
$ cd data-access/
```

2. 创建一个模块来管理，这个向导中的所有依赖

   运行`go mod init` ，并传递你的新代码的模块路径。

   ```bash
   $ go mod init example.com/data-access
   go: creating new go.mod: module example.com/data-access
   ```

   ```bash
   $ ls
   go.mod
   ```

   这个命令创建了`go.mod`文件, 然后你添加的依赖将会被列出在这个文件中，用于追踪这些依赖。获取依赖追踪更多的信息，请查看 [Managing dependencies](https://golang.org/doc/modules/managing-dependencies).

接下来，你将创建一个database

### 建立database

此节，将使用DBMS提供的工具创建database, table, 并添加数据。

此代码使用 [MySQL CLI](https://dev.mysql.com/doc/refman/8.0/en/mysql.html), 但是大多数DBMSes有自己的工具，也有相似的特性。

1. 打开终端

2. 登入DBMS, 以下为mysql示例

   ```bash
   root@1227f652bef0:~# mysql -h192.168.1.222 -uroot -p -P 32761
   MariaDB [(none)]> 
   ```

3. 在mariaDB命令提示处，创建一个数据库

   ```bash
   MariaDB [(none)]> create database recordings;
   ```

4. 使用文件编辑器，在data-access目录中，创建一个`create-tables.sql`文件，来保存sql脚本，它用来添加表。

5. 进入文件，粘贴以下sql代码，然后保存

   ```mysql
   DROP TABLE IF EXISTS album;
   CREATE TABLE album (
     id         INT AUTO_INCREMENT NOT NULL,
     title      VARCHAR(128) NOT NULL,
     artist     VARCHAR(255) NOT NULL,
     price      DECIMAL(5,2) NOT NULL,
     PRIMARY KEY (`id`)
   );
   
   INSERT INTO album 
     (title, artist, price) 
   VALUES 
     ('Blue Train', 'John Coltrane', 56.99),
     ('Giant Steps', 'John Coltrane', 63.99),
     ('Jeru', 'Gerry Mulligan', 17.99),
     ('Sarah Vaughan', 'Sarah Vaughan', 34.98);
   ```

   在这个sql代码，你:

   - 删除`album`表。如果想重新编辑此表，执行第1个命令，使得你之后重新运行这个脚本更容易。
   - 创建`album`表，此表有4个字段：`id`, `title`, `artist`, `price`. 每行的id值由DBMS自动的创建。
   - 添加3行记录

6. mysql命令行运行这个脚本

   使用source命令，以以下格式

   ```bash
   mysql> source /path/to/create-tables.sql
   ```

7. 在DBMS命令提示符中，使用`SELECT`语句验证成功添加表，并写入了数据。

   ```bash
   MariaDB [recordings]>  select * from album;
   +----+---------------+----------------+-------+
   | id | title         | artist         | price |
   +----+---------------+----------------+-------+
   |  1 | Blue Train    | John Coltrane  | 56.99 |
   |  2 | Giant Steps   | John Coltrane  | 63.99 |
   |  3 | Jeru          | Gerry Mulligan | 17.99 |
   |  4 | Sarah Vaughan | Sarah Vaughan  | 34.98 |
   +----+---------------+----------------+-------+
   4 rows in set (0.07 sec)
   ```

接下来，你将写go代码来连接，这样你就可以查询

### 查找并导入a database driver

现在已经有了mysql database和一些数据, 开始写代码

定位和导入一个翻译请求的数据库驱动。你通过`database/sql`包发起 数据库能理解的请求。

1. 访问 [SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers) wiki page ，查找你能使用的一个驱动。
   这个向导将会使用以下这个驱动

   - **MySQL**: https://github.com/go-sql-driver/mysql/ `[*]`

2. 注意，驱动包名为`github.com/go-sql-driver/mysql`

3. 使用文件编辑器，在上面[为你的代码创建一个目录](#为你的代码创建一个目录)一节中创建的目录中创建一个main.go文件。

4. 进入main.go文件，粘贴以下代码，来导入驱动package.

   ```go
   package main
   
   import "github.com/go-sql-driver/mysql"
   ```

   在这个代码中，你：

   - 添加你的代码到main包，因此你能独立执行main.go
   - 导入mysql驱动`github.com/go-sql-driver/mysql`

随着驱动的导入，你将可以开始写代码来访问数据库

### 获取一个数据库handle和connect

写go代码，通过数据库的handle访问数据库

你将使用指向 `sql.DB`数据结构的指针，它代表访问特定数据库

#### 写代码

1. `main.go`中，在Import下方，粘贴以下go代码，来创建数据库handle

   ```go
   var db *sql.DB
   
   func main() {
       // Capture connection properties.
       cfg := mysql.Config{
           User:   os.Getenv("DBUSER"),
           Passwd: os.Getenv("DBPASS"),
           Net:    "tcp",
           Addr:   "127.0.0.1:3306",
           DBName: "recordings",
       }
       // Get a database handle.
       var err error
       db, err = sql.Open("mysql", cfg.FormatDSN())
       if err != nil {
           log.Fatal(err)
       }
   
       pingErr := db.Ping()
       if pingErr != nil {
           log.Fatal(pingErr)
       }
       fmt.Println("Connected!")
   }
   ```

   在这个代码中，你：

   - 声明一个`*sql.DB`类型的`db`变量。这个就是你的数据库handle

     让`db`是全局变量简化了这个例子。在生产环境中，你应该避免全局变量。例如：传递变量给需要他的函数，或在数据结构中包装他。

   - Use the MySQL driver’s [`Config`](https://pkg.go.dev/github.com/go-sql-driver/mysql#Config) – and the type’s [`FormatDSN`](https://pkg.go.dev/github.com/go-sql-driver/mysql#Config.FormatDSN) -- 收集连接属性，并格式化为DSN连接字符串。

     `Config`数据结构比连接字符串更易读

   - 调用 `sql.Open` 初始化db变量，并传递`FormatDSN`返回值给它。

   - 检查其反回错误。例如：如果你的连接配置不对，就可能失败。

     简化代码，`log.Fatal`将会打印错误到控制台。生产代码中，你可能会使用更优雅的方式处理错误。

   - 调用 `DB.Ping` 确保数据库工作正常。在运行时，`sql.Open`并不会立即连接，现在使用`Ping` 可以确保在他需要连接时是`database/sql`包可以连接。
   - 检查来自`Ping`的错误，假如连接失败了。
   - 如果`Ping`成功地连接，打印消息。

2. `main.go`文件上面，package申明的下面，导入大量包，由你写的代码依赖的所有包。

   文件的顶端应该像以下这样

   ```go
   package main
   
   import (
       "database/sql"
       "fmt"
       "log"
       "os"
   
       "github.com/go-sql-driver/mysql"
   )
   ```

3. 保存main.go



#### 运行代码

1. 开始追踪mysql驱动为一个依赖。

   使用`go get`添加`github.com/go-sql-driver/mysql`模块作为你自己模块的依赖。使用`.`表示获取依赖是当前目录中所有代码的所有依赖。

   ```go
   $ export GOPROXY=https://goproxy.io,direct
   $ go get .
   ```

   以上代码中，你

   - [依赖管理](https://golang.org/doc/modules/managing-dependencies#proxy_server)中会说明代理配置

2. 命令行添加`DBUSER`, `DBPASS`环境变量，它们用来被go程序使用。

   on Linux or Mac

   ```bash
   $ export DBUSER=username
   $ export DBPASS=password
   ```

   on Windows

   ```powershell
   C:\Users\you\data-access> set DBUSER=username
   C:\Users\you\data-access> set DBPASS=password
   ```

3. 命令行处于包含main.go目录中，通过`go run .`运行代码，表示运行当前目录这个包

   ```bash
   go run .
   Connected!
   ```

   如果报错`[mysql] 2018/07/02 17:10:36 driver.go:120: could not use requested auth plugin 'mysql_native_password': this user requires mysql native password authentication.`, 请参考[github issue](https://github.com/go-sql-driver/mysql/issues/828), 其中说明当使用早期版本 v1.3.0 将不存在此问题

   参考[go 模块获取指定版本](https://golang.org/doc/modules/managing-dependencies#getting_version)

   解决此问题，可以先修改依赖

   ```bash
   $ go get github.com/go-sql-driver/mysql@v1.3.0
   go: downloading github.com/go-sql-driver/mysql v1.3.0
   go get: downgraded github.com/go-sql-driver/mysql v1.6.0 => v1.3.0
   
   ```

   ```bash
   $ cat go.mod
   module example.com/data-access
   
   go 1.16
   
   require github.com/go-sql-driver/mysql v1.3.0
   
   ```

   ```bash
   $ go run .
   Connected!
   ```

   现在可以发现，已经恢复正常

你能连接了！接下来将查询一些数据

### 查询多行

这篇中，你将使用go执行sql查询返回多行。

sql语句一般返回多行，将使用`database/sql`中的`Query`方法来遍历每一行。（你也将学习查询单行，在这篇 [Query for a single row](https://golang.org/doc/tutorial/database-access#single_row).)



#### 写代码

1. 在main.go中，在func main上面，粘贴以下`Album`数据结构。这个会保存查询返回的行数据

   ```go
   type Album struct {
       ID     int64
       Title  string
       Artist string
       Price  float32
   }
   ```

2. 在func main上方，粘贴以下 `albumsByArtist`函数来查询数据库

   ```go
   // albumsByArtist queries for albums that have the specified artist name.
   func albumsByArtist(name string) ([]Album, error) {
       // An albums slice to hold data from returned rows.
       var albums []Album
   
       rows, err := db.Query("SELECT * FROM album WHERE artist = ?", name)
       if err != nil {
           return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
       }
       defer rows.Close()
       // Loop through rows, using Scan to assign column data to struct fields.
       for rows.Next() {
           var alb Album
           if err := rows.Scan(&alb.ID, &alb.Title, &alb.Artist, &alb.Price); err != nil {
               return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
           }
           albums = append(albums, alb)
       }
       if err := rows.Err(); err != nil {
           return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
       }
       return albums, nil
   }
   ```

   在这个代码中，你：

   - 申明上面定义`Album`类型的`albums`slice。这个会保留返回所有行的数据。结构的字段名和类型与数据库的字段名和类型一致。

   - 使用 [`DB.Query`](https://pkg.go.dev/database/sql#DB.Query) 执行sql语句来查询album中指定artist名的行。

     `Query`的第1个参数是sql语句。后面参数可以是0个或多个任意类型的参数。这将在sql语句中替换这些参数指定的值。通过从参数值分离出sql语句（而不是串联），你启动`database/sql`包发送从sql文本分离的值，避免了sql注入的风险。

   - 推迟rows的关闭，这样行保持的所有资源将在函数退出时释放

   - 循环返回的行，通过`rows.Scan`将每行的字段值分配给`Album`数据结构字段。

     `Scan`接受go值指针的列表。这个地方字段值将写入。你传递指针给`alb`变量的字段，使用`&`操作符创建它。`Scan`通过指针更新结构字段。

   - 循环中，检查错误来自扫描字段值到结构字段的错误。
   - 循环中，追加新的`alb`到albums slice
   - 循环之后，使用`rows.Err`检查错误 来自总体查询的错误。注意如果query自身失败，检查这儿的错误是查找结果未完成的惟一方法。

3. 更新main函数，调用`albumsByArtist`

   在`func main`结尾处添加以下行

   ```go
   albums, err := albumsByArtist("John Coltrane")
   if err != nil {
       log.Fatal(err)
   }
   fmt.Printf("Albums found: %v\n", albums)
   ```

   新代码中，你

   - 调用`albumsByArtist`函数，分配返回值给albums变量
   - 打印结果

#### 运行代码

从命令行，运行于`main.go`文件所处目录中，运行这个代码：

```bash
$ go run .
Connected!
Albums found: [{1 Blue Train John Coltrane 56.99} {2 Giant Steps John Coltrane 6 3.99}]
```

### 查询单行

https://golang.org/doc/tutorial/database-access#single_row







