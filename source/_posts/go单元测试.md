---
title: go单元测试
date: 2020-09-16 22:54:43
tags:
    - golang
    - 测试
categories:
 - 后台开发
 - golang
comments: true
---
## go单元测试
 
### 1.什么是单元测试
 在计算机编程中，单元测试（英语：Unit Testing）又称为模块测试, 是针对程序模块（软件设计的最小单位）来进行正确性检验的测试工 作。程序单元是应用的最小可测试部件。在过程化编程中，一个单元就是单个程序、函数、过程等；对于面向对象编程，最小单元就是方法，包括基类（超类）、抽象类、或者派生类（子类）中的方法。这是维基百科的定义。单元测试的目的简单来说就是保证代码质量。
 
### 2.怎么做好单元测试

#### 1.结构设计需要做好分层，剥离依赖
 经验不足的编码人员，往往只关心流程与结果，不关注结构，很自然的就把模块鼓捣成一团浆糊式的代码块。浆糊式的代码块是很难做单测的。函数之间关系错综复杂，很多函数还依赖了不容易构造的条件，为了测试它们，你不得不多做很多额外的打桩工作。要让模块更易于单测，要对模块进行分层，每一层完成特定的功能，层与层之间通过数据结构交互 。尽可能采用一些相对简单、易于构造的数据结构作为函数之间的交互媒介。当需要对某个函数进行单测的时候，仅需构造这些相对简单的数据结构即可，而无需构造复杂的测试条件，尤其是协作模块、及物理环境。
    
#### 2. 函数设计需要单一、内聚、接口稳定
函数功能单一、内聚，功能明确。只通过参数/返回值与别的函数协作，这样的函数可理解性更好、可复用性也更好，即使模块的需求和设计发生变化，复用性好的函数，其接口变化也会更小。对这类函数进行单元测试也方便测试。
       
#### 3. 单测案例编写应当先行，减少调试时间
一些开发流程是先写代码-->调试代码-->补单测 。在编写实现代码之前写单测用例，因为这样可以节省大量的调试时间，而且有助于更好的理解函数的功能、输入输出，可以在这个过程中调整设计不佳的函数接口；流程转变一下 单测-->写代码-->单测验证代码
    
###  3. GO单元测试技术
 go语言本身就支持单测go test 命令将自动执行单测案例。GO语⾔单元测试对⽂件名和⽅法名、参数都有很严格的要求，约定如下
1. ⽂件名必须以xx_test.go命名 ，以_test.go结尾 
2. ⽅法必须是Test[^a-z]开头 
3. ⽅法参数必须是 t *testing.T

go单元测试是非常好做的.自带的testing包，配置断言库 goconvey。高端一点的打桩工具monkey。
#### 1.基本测试

1. 一个add函数
````go
package math
func Add(a, b int) int {
    return a + b
}
``````

单测代码

``` go
pachage math_test

import (
    "testing"
    . "github.com/smartystreets/goconvey/convey"
)


func TestAdd(T *testing.T) {
    res := Add(1, 2)
    Convey("now: \n", T, func() {
        So(res, ShouldEqual, 3)
    })
}
```

#### 2.对函数打桩

打桩含义是当测试的单元依赖一些外部条件，其他函数等。模拟一些失败情况。go里面比较好用的打桩工具为monkey。go get 
bou.ke/monkey 即可安装。

 下面将以例如说明如何打桩。
 
 ```go

func Now() time.Time {
    return time.Now()
}


func PrintYes() string {
    t := Now()
    if t.Format("15:03:04") == "12:12:12" {
        return "yes"
    }
    return "no"
}

```
Print函数功能为12:12:12秒时候输出一个hello world，如何测试这个函数的正确性？一般就是修改系统时间。
还有高端玩法，那就是打桩，我们发现Print函数依赖Now()函数，如果Now函数的返回能随意设置就好了。直接设置为12:12:12就ok了。
monkey对Now打桩

```go
package example
import (
    "testing"
    "bou.ke/monkey"
    . "github.com/smartystreets/goconvey/convey"
)


func TestPrintYes(T *testing.T) {
    monkey.Patch(Now, func() time.Time {
        t, _ := time.Parse("2006-01-02 15:04:05", "2018-12-01 12:12:12")
        return t
    })
    res := PrintYes()
    Convey("now: \n", T, func() {
        So(res, ShouldEqual, "yes")
    })
}

```
使用monkey.Patch 函数，对Now函数打桩。Now此时会返回"2018-12-01 12:12:12”的时间。此时就可以测试PrintYes函数,不用等12:12:12秒了。
#### 3.对对象打桩
```go

//Mymap xx
type Mymap struct {
    Mu sync.Mutex
    Data map[int]int
}
//New from
func New() *Mymap {
    var m Mymap
    m.Data = make(map[int]int)
    return &m
}
//Set k-v
func (M *Mymap) Set(key, val int) {
    defer M.Mu.Unlock()
    M.Mu.Lock()
    M.Data[key] = val
}
//Get k-v
func (M *Mymap) Get(key int) (int, bool) {
    defer M.Mu.Unlock()
    M.Mu.Lock()
    val, ok := M.Data[key]
    return val, ok
}

```
上面有一个Mymap对象，现在想要对对象的Get方法打桩。怎么做的呢
```go

func TestMap(T *testing.T) {
    var d *Mymap
    monkey.PatchInstanceMethod(reflect.TypeOf(d), "Get", func(*Mymap, int) (int, bool) {
        return 11, false
    })
    ret, ok := d.Get(123)
    Convey("now: \n", T, func() {
        So(ret, ShouldEqual, 11)
        So(ok, ShouldEqual, false)
    })
}

```
用PatchInstanceMethod方法即可.上面例子中，指定Get方法返回特定的值。


注：bou.ke/monkey 的原理为修改打桩函数的指令序列，进入函数后，直接让他跳转到执行打桩函数的序列。
使用monkey打桩我们可以做：

* 解决第三方系统，组件依赖问题
* 解决一些分支（异常）不容易走到

#### 4.代码覆盖率高单测越全面，
通过代码覆盖率查看哪些分支没有单测覆盖，[查看go代码覆盖率](https://studygolang.com/articles/10242)

 1. go test -coverprofile=coveragefile 查看覆盖率与生成覆盖率文件
 2. go tool cover -html=coveragefile 生成html文件

#### 5.tips
1.由于内联问题，编译优化。打桩可能失效。采用 go test -gcflags="-N -l" 解决。