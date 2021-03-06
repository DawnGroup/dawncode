
# 拜拜了，GOPATH君！新版本Golang的包管理入门教程
Go 1.11和1.12实现了对包管理的初步支持，Go的新依赖管理系统使依赖版本信息明确且易于管理。
[Using Go Modules - The Go Blog](https://blog.golang.org/using-go-modules)

![](/asset/pic/1.png)


## 新的包管理模式有什么不同？
作为Go语言的推广者，常常被问到各种关于Go语言的基础特性问题。
其中，关于包管理方面的问题会让我**非常尴尬**，因为Go的包管理在1.11之前与Python、Node、Java比较起来真的只能算是**“仅仅可用”而已**。

因为：

1. 在不使用额外的工具的情况下，Go的依赖包需要手工下载，
2. 第三方包没有版本的概念，如果第三方包的作者做了不兼容升级，会让开发者很难受
3. 协作开发时，需要统一各个开发成员本地$GOPATH/src下的依赖包
4. 引用的包引用了已经转移的包，而作者没改的话，需要自己修改引用。
5. 第三方包和自己的包的源码都在src下，很混乱。对于混合技术栈的项目来说，目录的存放会有一些问题

新的包管理模式解决了以上问题
1. 自动下载依赖包
2. 项目不必放在GOPATH/src内了
3. 项目内会生成一个go.mod文件，列出包依赖
4. 所以来的第三方包会准确的指定版本号
5. 对于已经转移的包，可以用replace 申明替换，不需要改代码


## 开始吧用一个小例子来说明问题！
## 准备工作
1. 升级golang 版本到 1.12  [Go下载](https://golang.org/dl/)
2. 添加环境变量 **GO111MODULE** 为 **on** 或者**auto**
``` shell
GO111MODULE=auto
```

**准备完毕，非常简单吧！！**

## 创建一个项目

首先，在`$GOPATH/src`路径外的你喜欢的地方创建一个目录，cd 进入目录，新建一个hello.go文件，内容如下
``` go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, world!")
}

``` 

## 初始化模块
在当前目录下，命令行运行 go mod init + 模块名称 初始化模块
``` shell
go mod init hello
```

运行完后，会在当前项目目录下生成一个**go.mod** 文件，这是一个关键文件，之后的包的管理都是通过这个文件管理。

> 官方说明：除了go.mod之外，go命令还维护一个名为go.sum的文件，其中包含特定模块版本内容的预期加密哈希  
> go命令使用go.sum文件确保这些模块的未来下载检索与第一次下载相同的位，以确保项目所依赖的模块不会出现意外更改，无论是出于恶意、意外还是其他原因。 go.mod和go.sum都应检入版本控制。  
**go.sum** 不需要手工维护，所以可以不用太关注。


生成出来的文件包含模块名称和当前的go版本号
```
module hello

go 1.12
```

**注意**：子目录里是不需要init的，所有的子目录里的依赖都会组织在根目录的go.mod文件里

## 来吧，搞点小事情，看看go.mod怎么工作的
接下来，让你的项目依赖一下第三方包
以大部分人都熟悉的beego为例吧！
修改Hello.go文件：
```go
package main

import "github.com/astaxie/beego"

func main() {
    beego.Run()
}

```

按照过去的做法，要运行hello.go需要执行`go get` 命令 下载beego包到 $GOPATH/src

**但是，使用了新的包管理就不在需要这样做了**

直接  `go run hello.go`

稍等片刻… go 会自动查找代码中的包，下载依赖包，并且把具体的依赖关系和版本写入到go.mod和go.sum文件中。
查看go.mod，它会变成这样：
```
module hello

go 1.12

require github.com/astaxie/beego v1.11.1

```

`require ` 关键子是引用，后面是包，最后v1.11.1 是引用的版本号

这样，一个使用Go包管理方式创建项目的小例子就完成了。

那么，接下来，几个小问题来了

### 问题一：依赖的包下载到哪里了？还在GOPATH里吗？

**不在。**
使用Go的包管理方式，依赖的第三方包被下载到了`$GOPATH/pkg/mod`路径下。

如果你成功运行了本例，可以在您的`$GOPATH/pkg/mod` 下找到一个这样的包 `github.com/astaxie/beego@v1.11.1`

### 问题二： 依赖包的版本是怎么控制的？

在上一个问题里，可以看到最终下载在`$GOPATH/pkg/mod` 下的包 `github.com/astaxie/beego@v1.11.1` 最后会有一个版本号 1.11.1，也就是说，`$GOPATH/pkg/mod`里可以保存相同包的不同版本。

版本是在go.mod中指定的。

如果，在go.mod中没有指定，go命令会自动下载代码中的依赖的最新版本，本例就是自动下载最新的版本。

如果，在go.mod用require语句指定包和版本 ，go命令会根据指定的路径和版本下载包，
指定版本时可以用`latest`，这样它会自动下载指定包的最新版本；

**依赖包的版本号是什么？** 是包的发布者标记的版本号，格式为 vn.n.n (n代表数字)，本例中的beego的历史版本可以在其代码仓库release看到[Releases · astaxie/beego · GitHub](https://github.com/astaxie/beego/releases)

如果包的作者还没有标记版本，默认为 v0.0.0


### 问题三： 可以把项目放在$GOPATH/src下吗？

**可以。**
但是go会根据GO111MODULE的值而采取不同的处理方式
默认情况下，`GO111MODULE=auto` 自动模式

- **auto** 自动模式下，项目在`$GOPATH/src`里会使用`$GOPATH/src`的依赖包，在$GOPATH/src外，就使用go.mod 里 require的包
- **on** 开启模式，1.12后，无论在`$GOPATH/src`里还是在外面，都会使用go.mod 里 require的包
- **off** 关闭模式，就是老规矩。

### 问题三： 依赖包中的地址失效了怎么办？比如 golang.org/x/… 下的包都无法下载怎么办？

在go快速发展的过程中，有一些依赖包地址变更了。
以前的做法
1. 修改源码，用新路径替换import的地址
2. git clone 或 go get 新包后，copy到$GOPATH/src里旧的路径下
无论什么方法，都不便于维护，特别是多人协同开发时。

使用go.mod就简单了，在go.mod文件里用 replace 替换包，例如

`replace golang.org/x/text => github.com/golang/text latest`

这样，go会用 github.com/golang/text 替代golang.org/x/text，原理就是下载github.com/golang/text 的最新版本到 `$GOPATH/pkg/mod/golang.org/x/text`下。

### 问题四： init生成的go.mod的模块名称有什么用？

本例里，用 go mod init hello 生成的go.mod文件里的第一行会申明
`module hello`

因为我们的项目已经不在`$GOPATH/src`里了，那么引用自己怎么办？就用模块名+路径。

例如，在项目下新建目录 utils，创建一个tools.go文件:
``` go
package utils

import “fmt”

func PrintText(text string) {
	fmt.Println(text)
}
```


在根目录下的hello.go文件就可以 import “hello/utils”   引用utils

``` go
package main

import (
"hello/utils"

"github.com/astaxie/beego"
)

func main() {

	utils.PrintText("Hi")

	beego.Run()
}


```

### 问题五：以前老项目如何用新的包管理

1. 如果用auto模式，把项目移动到`$GOPATH/src`外
2. 进入目录，运行 `go mod init + 模块名称`
3. `go build` 或者 `go run` 一次



**关于Go 1.12的包管理介绍大致就到此了**

根据官方的说法，从Go 1.13开始，模块管理模式将是Go语言开发的默认模式。

**所以，Pick起来吧！**

有问题或者需要讨论的朋友，可以给我留言，**共同学习，一起进步**

#Writing