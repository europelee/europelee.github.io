# Using Go Modules(译)
本篇文章来自<https://blog.golang.org/using-go-modules>.
## 介绍
Go 从 1.11 and 1.12 开始对modules有了初步支持，modules是Go新的依赖包管理工具，它能让我们对依赖包版本信息可以做明确和更容易管理。本篇文章是对使用modules基本操作的初步介绍，后续文章将介绍modules的其他使用场景。  
  
一个module是一个Go包的集合，使用一个在项目根目录上叫go.mod的文件做管理。go.mod定义了module的module path,作为根目录的导入路径，并且还定义了项目的依赖包需求，每个依赖包需求通过module path和指定的语义版本号来定义。  

从Go 1.11开始，go命令支持modules的使用，只要当前目录或任何父目录包含一个go.mod文件，即使项目目录不在\$GOPATH/src路径（为了兼容旧版本，go命令对于在$GOPATH/src的项目仍然使用GOPATH模式，即使有go.mod文件）。从1.13开始，module模式将会是默认的开发配置。  

本篇文章将会以下面的顺序来描述使用modules来进行go代码开发的一系列操作：  
* 创建一个新的module
* 添加一个依赖包  
* 升级依赖包  
* 添加一个带新主版本号的依赖包
* 升级一个依赖包到新的主版本号
* 删除未使用的依赖包  

## 创建一个新的module  

在非$GOPATH/src路径上创建一个新的空目录，然后进入该目录，创建hello.go:  
```go
package hello

func Hello() string {
    return "Hello, world."
}
```
然后再创建一个测试源代码hello_test.go:  
```go
package hello

import "testing"

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}
```
该项目目录现在并未包含一个module，因为没有go.mod的文件在项目目录下，假设我们是创建的目录是在/home/gopher/hello，进入该目录执行go test, 我们会看到如下结果：  
> $ go test  
> PASS  
> ok  _/home/gopher/hello    0.020s  
> $  

最后一行总结了所有包测试结果。由于hello项目目录不在$GOPATH路径下，也没有在任何module下，导致go命令不知道任何导入路径，只能基于目录名字生成了一个名字: _/home/gopher/hello.  
让我们使用go mod init为当前目录配置moduleo，然后再go test:  
> $ go mod init example.com/hello  
> go: creating new go.mod: module example.com/hello  
> $ go test  
> PASS  
> ok      example.com/hello    0.020s  
> $  

这样，你就完成并测试了你的第一个module.  
go mod init命令生成了一个go.mod文件:  
> $ cat go.mod  
> module example.com/hello  
> go 1.12  
> $  

go.mod只会出现在module的根路径下(译注：sub module除外)。在子目录下的包要正确导入，path路径需要由两部分组成：module path+子目录路径。举例，假如我们创建了一个world子目录，我们不需要在world子目录下执行go mod init命令, world包会自动被识别作为 example.com/hello 的一部分，可以使用 example.com/hello/world 导入该world包。 

## 添加一个依赖包  
Go modules的引入的主要动机是改进代码重用（即添加依赖包）的体验。  
更新hello.go，导入 rsc.io/quote 包：  
```go
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
```
运行go test, 结果：  
> $ go test  
> go: finding rsc.io/quote v1.5.2  
> go: downloading rsc.io/quote v1.5.2  
> go: extracting rsc.io/quote v1.5.2  
> go: finding rsc.io/sampler v1.3.0  
> go: finding golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c  
> go: downloading rsc.io/sampler v1.3.0  
> go: extracting rsc.io/sampler v1.3.0  
> go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c  
> go: extracting golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c  
> PASS  
> ok      example.com/hello    0.023s  
> $  

go命令使用在go.mod文件里面指定的依赖包module版本来解析依赖包的导入。当它碰到一个go.mod的任何module没有提供的包导入，go命令会自动查找包含该包的module，并且将它添加到go.mod, 使用最新版本。在我们例子中，go test会解析new import rsc.io/quote 到module rsc.io/quote v1.5.2. 而且会自动下载 rsc.io/quote 的两个依赖包. 但只有直接依赖包会被记录在go.mod文件：  
> $ cat go.mod  
> module example.com/hello  
> go 1.12  
> require rsc.io/quote v1.5.2  
> $  

第二次go test不会重复上面的过程，因为go.mod已经更新了依赖包需求，并且被下载的modules已经被缓存在本地路径$GOPATH/pkg/mod：  
> $ go test  
> PASS  
> ok      example.com/hello    0.020s  
> $  

go命令可以快速容易添加新的依赖包，这不是没有代价的。在正确性、安全性、适合的licensing等方面都依赖于依赖包，更多的思考，参考Russ Cox's blog[Our Software Dependency Problem](https://research.swtch.com/deps).  
正如我们所看，添加一个直接的依赖包会带来另外的间接依赖包，go list -m all会罗列出当前的module和它所有的依赖:  
> $ go list -m all  
> example.com/hello  
> golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c  
> rsc.io/quote v1.5.2  
> rsc.io/sampler v1.3.0  
> $

当前的module，即主module, 总是处在第一行，接下是它的依赖包。  
golang.org/x/text version v0.0.0-20170915032832-14c0d48ead0c是一个伪版本号的例子，go命令为一个untagged的commit设置的版本语法。  
除了go.mod, go命令也会维护一个go.sum的文件，里面包含了那些指定的module版本内容的加密哈希。  
> $ cat go.sum  
> golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZO...
> golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:Nq...  
> rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3...  
> rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPX...  
> rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/Q...  
> rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9...  
> $  

go命令使用go.sum文件来确保这些modules的下载会同第一下载的内容一模一样，保证项目所依赖的modules没有任何意外的改变，go.mod和go.sum都需要被提交做版本控制。  

## 升级依赖包  
Go modules使用语义版本，由3个部分组成：major, minor, 和patch. 例如 v0.1.2, major版本号是0, minor是1, patch是2. 我们先讲下minor版本号升级。下节, 我们再来考虑major的版本升级。  

当前我们使用的 golang.org/x/text 是untagged的版本，升级下text到最新的tagged版本，测试程序是否仍正确:  
> $ go get golang.org/x/text  
> go: finding golang.org/x/text v0.3.0  
> go: downloading golang.org/x/text v0.3.0  
> go: extracting golang.org/x/text v0.3.0  
> $ go test  
> PASS  
> ok      example.com/hello    0.013s  
> $

测试通过，使用go list -m all，并查看go.mod文件：  
> $ go list -m all
> example.com/hello  
> golang.org/x/text v0.3.0  
> rsc.io/quote v1.5.2  
> rsc.io/sampler v1.3.0  
> $ cat go.mod  
> module example.com/hello  
> go 1.12  
> require (  
>     golang.org/x/text v0.3.0 // indirect  
>     rsc.io/quote v1.5.2  
> )  
> $


The golang.org/x/text package 已经升级到最新的tagged version (v0.3.0). The go.mod file也更新到了v0.3.0. The indirect comment indicates a dependency is not used directly by this module, only indirectly by other module dependencies. See go help modules for details.  
升级 rsc.io/sampler minor版本. 同上面步骤一样，执行go get and running tests:  
> $ go get rsc.io/sampler  
> go: finding rsc.io/sampler v1.99.99  
> go: downloading rsc.io/sampler v1.99.99  
> go: extracting rsc.io/sampler v1.99.99  
> $ go test  
> --- FAIL: TestHello (0.00s)  
>     hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of  
> beer, ...", want "Hello, world."  
> FAIL  
> exit status 1  
> FAIL    example.com/hello    0.014s  
> $  

Uh, oh! 测试结果表明 rsc.io/sampler 的最新版本不能兼容我们的项目. Let's list the available tagged versions of that module:  
> $ go list -m -versions rsc.io/sampler  
> rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99  
> $

We had been using v1.3.0; v1.99.99 is clearly no good. Maybe we can try using v1.3.1 instead:
> $ go get rsc.io/sampler@v1.3.1  
> go: finding rsc.io/sampler v1.3.1  
> go: downloading rsc.io/sampler v1.3.1  
> go: extracting rsc.io/sampler v1.3.1  
> $ go test  
> PASS  
> ok      example.com/hello    0.022s  
> $

通过在go get 显示地使用@v1.3.1来获取指定版本. 默认是@latest。  

## 添加一个带新主版本号的依赖包
我们在hello包添加一个新的函数Proverb，该函数通过调用module rsc.io/quote/v3 的一个方法quote.Concurrency来返回一个Go concurrency：  
```go
package hello

import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
    return quote.Hello()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

Then we add a test to hello_test.go:  
```go
func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}
```
Then we can test our code:  
```
$ go test
go: finding rsc.io/quote/v3 v3.1.0
go: downloading rsc.io/quote/v3 v3.1.0
go: extracting rsc.io/quote/v3 v3.1.0
PASS
ok      example.com/hello    0.024s
$
```
通过go list,我们注意到我们的主module同时依赖quote两个版本：    
```
$ go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
$
```
一个Go module对于不同的主版本号(v1, v2等)使用不同的module path: 从v2开始，路径必须以主版本号作为结束。In the example, rsc.io/quote 的v3版本的路径不再是 rsc.io/quote: instead, it is identified by the module path rsc.io/quote/v3. 这种约束可以称为语义导入版本，Go module给那些不兼容的包(those with different major versions)不同的名字. 相反, v1.6.0 of rsc.io/quote 必须向后兼容v1.5.2。  
go命令仅允许一个主版本号的module最多只能有一个版本号，即一个程序不能同时使用rsc.io/quote v1.5.2 和 rsc.io/quote v1.6.0来编译。但是允许module可以同时使用两个不同的主版本号，使得module的调用方可以渐进升级这个module的版本。  
## 升级一个依赖包到新的主版本号
现在我们要将 rsc.io/quote 彻底转为 rsc.io/quote/v3 的使用。由于主版本号的改变，我们应该知道某些APIs会被删除，重命名，或者不兼容:  
```
$ go doc rsc.io/quote/v3
package quote // import "rsc.io/quote"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
$
```
(There is also a [known bug](https://golang.org/issue/30778) in the output; the displayed import path has incorrectly dropped the /v3.)

We can update our use of quote.Hello() in hello.go to use quoteV3.HelloV3():  
```go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}
```
Let's re-run the tests to make sure everything is working:  
```
$ go test
PASS
ok      example.com/hello       0.014s
```
## 删除未使用的依赖包
如果我们删除所有对 rsc.io/quote 的使用, but it still shows up in go list -m all and in our go.mod file:  
```
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote v1.5.2
    rsc.io/quote/v3 v3.0.0
    rsc.io/sampler v1.3.1 // indirect
)
$
```
为啥呢？因为编译一个单一的包，像go build, go test, 很容易知道某些module是不存在并需要被添加，但不清楚什么时候可以被安全删除。只能在检查module里面的所有包以及可能的build tag combinations for those packages才能删除一个依赖包，An ordinary build command does not load this information, and so it cannot safely remove dependencies.

go mod tidy command用于清除unused dependencies:
```
$ go mod tidy
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1 // indirect
)

$ go test
PASS
ok      example.com/hello    0.020s
$
```
## Conclusion

Go modules are the future of dependency management in Go. Module functionality is now available in all supported Go versions (that is, in Go 1.11 and Go 1.12).

This post introduced these workflows using Go modules:

* go mod init creates a new module, initializing the go.mod file that describes it.
* go build, go test, and other package-building commands add new dependencies to go.mod as needed.
* go list -m all prints the current module’s dependencies.
* go get changes the required version of a dependency (or adds a new dependency).
* go mod tidy removes unused dependencies.

We encourage you to start using modules in your local development and 添加 go.mod and go.sum这两个文件到你的项目里面. To provide feedback and help shape the future of dependency management in Go, please send us bug reports or experience reports.

Thanks for all your feedback and help improving modules.

By Tyler Bui-Palsulich and Eno Compton