[TOC]

# 02 快速开始

​		这个程序实现的功能很常见，能在很多现在开发的 Go 程序里发现类似的功能。这个程序从不同的数据源拉取数据，将数据内容与一组搜索项做对比，然后将匹配的内容显示在终端窗口。这个程序会读取文本文件，进行网络调用，解码 XML 和 JSON 成为结构化类型数据，并且利用 Go 语言的并发机制保证这些操作的速度。

代码地址：chapter2/sample

![02程序架构流程图](./markdownimage/02程序架构.png)

## 程序项目与结构目录

```
- sample
  - data
     data.json -- 包含一组数据源
   - matchers
     rss.go -- 搜索 rss 源的匹配器
   - search
     default.go -- 搜索数据用的默认匹配器
     feed.go -- 用于读取 json 数据文件
     match.go -- 用于支持不同匹配器的接口
     search.go -- 执行搜索的主控制逻辑
   main.go -- 程序的入口
```

## main包

```go
package main

import (
	"log"
	"os"

	_ "github.com/goinaction/code/chapter2/sample/matchers"
	"github.com/goinaction/code/chapter2/sample/search"
)

// init is called prior to main.
func init() {
	// Change the device for logging to stdout.
	log.SetOutput(os.Stdout)
}

// main is the entry point for the program.
func main() {
	// Perform the search for the specified term.
	search.Run("president")
}

```

​		现在，只要简单了解以下内容：一个包定义一组编译过的代码，包的名字类似命名空间，可以用来间接访问包内声明的标识符。

​		这个特性可以把不同包中定义的同名标识符区别开。

​		**"import"**，顾名思义就是导入一段代码，让用户可以访问里面的标识符，例如类型、函数、常量和接口，比如main函数里面就引用了 search 包里面的 Run 函数。程序还分别导入了 log 和 os 包。  在第 7 行，导入前有个下划线：

```go
     _ "github.com/goinaction/code/chapter2/sample/matchers"
```

​		**这个技术是为了让 Go 语言对包做初始化操作，但是并不使用包里的标识符**。为了让程序的可读性更强，Go 编译器不允许声明导入某个包却不使用。下划线让编译器接受这类导入，并且调用对应包内的所有代码文件里定义的 init 函数。对这个程序来说，这样做的目的是调用 matchers 包中的 rss.go 代码文件里的 init 函数，注册 RSS 匹配器，以便后用。

​		代码中也有一个 init 函数，每个文件的 init 函数都会在 main 函数执行前调用。



## search包

​		这个程序使用的框架和业务逻辑都在 search 包里。这个包由 4 个不同的代码文件组成，每个文件对应一个独立的职责。我们会逐步分析这个程序的逻辑，到时再说明各个代码文件的作用。

​		由于整个程序都围绕**匹配器**来运作，我们先简单介绍一下什么是匹配器。这个程序里的匹配器，是指包含特定信息、用于处理某类数据源的实例。

​		在这个示例程序中有两个匹配器。框架本身实现了一个无法获取任何信息的默认匹配器，而在 matchers 包里实现了 RSS 匹配器。RSS 匹配器知道如何获取、读入并查找 RSS 数据源。随后我们会扩展这个程序，加入能读取 JSON 文档或 CSV 文件的匹配器。我们后面会再讨论如何实现匹配器。

### search.go

文件代码：

```go
package search

import (
	"log"
	"sync"
)

// A map of registered matchers for searching.
var matchers = make(map[string]Matcher)

// Run performs the search logic.
func Run(searchTerm string) {
	// Retrieve the list of feeds to search through.
	feeds, err := RetrieveFeeds()
	if err != nil {
		log.Fatal(err)
	}

	// Create an unbuffered channel to receive match results to display.
	results := make(chan *Result)

	// Setup a wait group so we can process all the feeds.
	var waitGroup sync.WaitGroup

	// Set the number of goroutines we need to wait for while
	// they process the individual feeds.
	waitGroup.Add(len(feeds))

	// Launch a goroutine for each feed to find the results.
	for _, feed := range feeds {
		// Retrieve a matcher for the search.
		matcher, exists := matchers[feed.Type]
		if !exists {
			matcher = matchers["default"]
		}

		// Launch the goroutine to perform the search.
		go func(matcher Matcher, feed *Feed) {
			Match(matcher, feed, searchTerm, results)
			waitGroup.Done()
		}(matcher, feed)
	}

	// Launch a goroutine to monitor when all the work is done.
	go func() {
		// Wait for everything to be processed.
		waitGroup.Wait()

		// Close the channel to signal to the Display
		// function that we can exit the program.
		close(results)
	}()

	// Start displaying results as they are available and
	// return after the final result is displayed.
	Display(results)
}

// Register is called to register a matcher for use by the program.
func Register(feedType string, matcher Matcher) {
	if _, exists := matchers[feedType]; exists {
		log.Fatalln(feedType, "Matcher already registered")
	}

	log.Println("Register", feedType, "matcher")
	matchers[feedType] = matcher
}
```

我们来逐步分析

```go
01  package search
02
03  import (
04  "log"
05  "sync"
06  )
07
08  // 注册用于搜索的匹配器的映射
09  var matchers = make(map[string]Matcher)
```

​		可以看到，每个代码文件都以 package 关键字开头，随后跟着包的名字。文件夹 search 下的每个代码文件都使用 search 作为包名。第 03 行到第 06 行代码导入标准库的 log 和 sync 包。

​		与第三方包不同，从标准库中导入代码时，只需要给出要导入的包名。编译器查找包的时候，总是会到 GOROOT 和 GOPATH 环境变量引用的位置去查找。

声明一个变量：

```go
08  // 注册用于搜索的匹配器的映射
09  var matchers = make(map[string]Matcher)
```

​		这个变量没有定义在任何函数作用域内，所以会被当成包级变量。这个变量使用关键字 var 声明，而且声明为 Matcher 类型的映射（map），这个映射以 string 类型值作为键，Matcher 类型值作为映射后的值。Matcher 类型在代码文件 matcher.go 中声明，后面再讲这个类型的用途。这个变量声明还有一个地方要强调一下：**<u>变量名 matchers 是以小写字母开头的。</u>**

​		在 Go 语言里，标识符要么从包里公开，要么不从包里公开**<u>。当代码导入了一个包时，程序可以直接访问这个包中任意一个公开的标识符。这些标识符以大写字母开头。以小写字母开头的标识符是不公开的，不能被其他包中的代码直接访问。</u>**但是，其他包可以间接访问不公开的标识符。例如，一个函数可以返回一个未公开类型的值，那么这个函数的任何调用者，哪怕调用者不是在这个包里声明的，都可以访问这个值。

​		这行变量声明还使用赋值运算符和特殊的内置函数 make 初始化了变量，<u>map 是 Go 语言里的一个引用类型，需要使用 make 来构造。</u>如果不先构造 map 并将构造后的值赋值给变量，会在试图使用这个 map 变量时收到出错信息。这是因为 map 变量默认的零值是 nil。

​		在 Go 语言中，所有变量都被初始化为其零值。对于数值类型，零值是 0；
​			对于字符串类型，零值是空字符串；对于布尔类型，零值是 false；
​			对于指针，零值是 nil。
​			对于引用类型来说，所引用的底层数据结构会被初始化为对应的零值。
​		但是被声明为其零值的引用类型的变量，会返回 nil 作为其值。

```go
11 // Run 执行搜索逻辑
12 func Run(searchTerm string) {
13 	// 获取需要搜索的数据源列表
14 	feeds, err := RetrieveFeeds()
15 	if err != nil {
16 		log.Fatal(err)
17 	} 
18
19 	// 创建一个无缓冲的通道，接收匹配后的结果
20 	results := make(chan *Result)
21
22 	// 构造一个 waitGroup，以便处理所有的数据源
23 	var waitGroup sync.WaitGroup
24
25 	// 设置需要等待处理
26 	// 每个数据源的 goroutine 的数量
27 	waitGroup.Add(len(feeds))
28
29 	// 为每个数据源启动一个 goroutine 来查找结果
30 	for _, feed := range feeds { 
31 		// 获取一个匹配器用于查找
32 		matcher, exists := matchers[feed.Type]
33 		if !exists {
34 			matcher = matchers["default"]
35 		} 
36
37 		// 启动一个 goroutine 来执行搜索
38 		go func(matcher Matcher, feed *Feed) {
39 			Match(matcher, feed, searchTerm, results)
40 			waitGroup.Done()
41 		}(matcher, feed)
42 	} 
43
44 	// 启动一个 goroutine 来监控是否所有的工作都做完了
45 	go func() {
46 		// 等候所有任务完成
47 		waitGroup.Wait()
48
49 		// 用关闭通道的方式，通知 Display 函数
50 		// 可以退出程序了
51 		close(results)
52 	}()
53
54 	// 启动函数，显示返回的结果，并且
55 	// 在最后一个结果显示完后返回
56 	Display(results)
57 }
```

先来看看 Run 函数是怎么定义的，如

```go
11 // Run 执行搜索逻辑
12 func Run(searchTerm string) {
```

​		Go 语言<u>使用关键字 func 声明函数，关键字后面紧跟着函数名、参数以及返回值</u>。对于 Run 这个函数来说，只有一个参数，是 string 类型的，名叫 searchTerm。这个参数是 Run 函数要搜索的搜索项。



```go
13 // 获取需要搜索的数据源列表
14 	feeds, err := RetrieveFeeds()
15 	if err != nil {
16 		log.Fatal(err)
17 	}
```

​		第 14 行调用了 search 包的 RetrieveFeeds 函数。这个函数返回两个值。第一个返回值是一组 Feed 类型的切片。**<u>切片是一种实现了一个动态数组的引用类型</u>**。在 Go 语言里可以用切片来操作一组数据。

​		第二个返回值是一个错误值。在第 15 行，检查返回的值是不是真的是一个错误。如果真的发生错误了，就会调用 log 包里的 Fatal 函数。Fatal 函数接受这个错误的值，并将这个错误在终端窗口里输出，随后终止程序。

​		不仅仅是Go语言，很多语言都允许一个函数返回多个值。一般会像 RetrieveFeeds 函数这样声明一个函数返回一个值和一个错误值。如果发生了错误，永远不要使用该函数返回的另一个值 ①这里可以看到简化变量声明运算符（:=）。这个运算符用于声明一个变量，同时给这个变量。这时必须忽略另一个值，否则程序会产生更多的错误，甚至崩溃。

​		这里可以看到简化变量声明运算符（:=）。这个运算符用于声明一个变量，同时给这个变量赋予初始值。
​		例如:    `ss := "hello"`    相当于： 

```go
	var ss string 
	ss = "hello"
```



```go
19  // 创建一个无缓冲的通道，接收匹配后的结果
20  results := make(chan *Result)
```

​		在第 20 行，使用内置的 make 函数创建了一个无缓冲的通道。根据经验，**<u>如果需要声明初始值为零值的变量，应该使用 var 关键字声明变量；如果提供确切的非零值初始化变量或者使用函数返回值创建变量，应该使用简化变量声明运算符。</u>**

​		在 Go 语言中，通道（channel）和映射（map）与切片（slice）一样，也是引用类型，不过通道本身实现的是一组带类型的值，这组值用于在 goroutine 之间传递数据。通道内置同步机制，从而保证通信安全。























