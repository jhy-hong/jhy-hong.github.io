# golang中的测试分类

golang中使用go test进行相应测试，目前有三种：

- 单元测试：对软件中**最小可测试单元进行检查和验证**，比如对一个函数的测试

- 性能测试：也称为基准测试，可测试<mark>一段程序的性能，可以得到时间消耗，内存使用情况</mark>的报告

- 示例测试：用于展示某个包或某个方法的用法

## 单元测试

目录结构

<img width="205" alt="Image" src="https://github.com/user-attachments/assets/eb5fa6e6-eb15-4f64-83a7-1268bb5b558a" />

源代码文件unit.go：

```go
package unitTesting

func Add(a, b int) int {
    return a + b
}
```

测试文件 unit_testing.go

```go
package unitTesting_test

import (
    "testing"
    "unitTesting"
)

func TestAdd(t *testing.T) {
    var a, b = 1, 2
    expected := 3
    actual := unitTesting.Add(a, b)
    if actual != expected {
        t.Errorf("error")
    }
}
```

测试文件属于unitTesting_test包，测试文件也可以和源文件在同一个包中，但常见的做法是创建一个包专门用于测试，这样<mark>可以使测试文件和源文件隔离</mark>，规则是在原包名加上 <mark>_test</mark>

**小结：**

- 测试文件名必须以<mark>_test.go</mark>结尾

- 测试函数名必须以<mark>TestXxx</mark>开始

- 在命令行下使用<mark>go test</mark>即可开启测试

## 基准测试

*目录结构*

![Image](https://github.com/user-attachments/assets/168950e5-21e7-44d5-a653-7e89249df538)

源代码文件 benchmakr.go

```go
package benchmarkTest

// MakeSliceWithoutAlloc 不预分配
func MakeSliceWithoutAlloc() []int {
    var newSlice []int
    for i := 0; i < 100000; i++ {
        newSlice = append(newSlice, i)
    }
    return newSlice
}

// MakeSliceWithAlloc 预分配
func MakeSliceWithAlloc() []int {
    var newSlice []int
    newSlice = make([]int, 0, 100000)
    for i := 0; i < 100000; i++ {
        newSlice = append(newSlice, i)
    }
    return newSlice
}
```

测试文件benchmark_test.go

```go
package benchmarkTest_test

import (
    "benchmarkTest"
    "testing"
)

func BenchmarkMakeSliceWithoutAlloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        benchmarkTest.MakeSliceWithoutAlloc()
    }
}
func BenchmarkMakeSliceWithAlloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        benchmarkTest.MakeSliceWithAlloc()
    }
}
```

性能测试函数的命名规则为BenchmarkXxx，其中Xxx为自定义的标识，需要以大写字母开头，通常为待测函数

testing.B提供了一系列用于辅助性能测试的方法或成员，如上面的b.N表示循环执行的次数，而N值不用程序员关心，N值是动态调整的，直到可靠的算出程序执行时间后才会停止。具体执行次数会在执行结束后打印出来。

**执行测试**

![Image](https://github.com/user-attachments/assets/ce226c0c-9a36-4bb2-a2f0-f8274a3b2b50)

-bench为go test的flag，该flag表示go test进行性能测试

**小结**

- 文件名必须以_test.go结尾

- 函数名必须以BenchmarkXxx开始

- 使用go test -bench=.命令可开始性能测试

## 示例测试

*目录结构*

![Image](https://github.com/user-attachments/assets/9b3cd7ae-5b8d-404e-8170-6cd19b033d2d)

源代码文件 example.go

```go
package exampleTest

import "fmt"

// 单行输出
func SayHello() {
    fmt.Println("Hello World")
}

// 两行顺序打印
func SyaGoodbye() {
    fmt.Println("hello")
    fmt.Println("Goodbye")

}

// 多行无序打印
func PrintNames() {
    students := make(map[int]string, 4)
    students[1] = "Jim"
    students[2] = "Tom"
    students[3] = "Bob"
    students[4] = "Jerry"
    for _, s := range students {
        fmt.Println(s)
    }
}
```

测试文件example_test.go：

```go
package exampleTest_test

import "exampleTest"

func ExampleSayHello() {
    exampleTest.SayHello()
    // OutPut: Hello World
}
func ExampleSyaGoodbye() {
    exampleTest.SyaGoodbye()
    // OutPut:
    // hello
    // Goodbye
}
func ExamplePrintNames() {
    exampleTest.PrintNames()
    // Unordered output:
    // Jim
    // Tom
    // Bob
    // Jerry
}
```

**<mark>小结</mark>**：

- 示例测试函数名必须以Example开头

- 检测单行输出格式为// Output:<期望字符串>

- 检测多行输出格式为// Output:  \  <期望字符串>  \ <期望字符串>,每个期望字符占一行

- 检测无序输出格式为// Unordered output:\ <期望字符串> \ <期望字符串>,每个期望字符占一行

- 测试字符串会自动忽略字符串前后的空白字符

- 如果测试函数中没有Output标识，则该函数不会执行

- 执行测试可以使用go test此时该目录下的其他测试文件也会一并被执行

- 执行测试可以使用go test  < xxx_test.go > ，此时仅执行特定文件中的测试函数

## 测试参数

go test包含丰富的参数，一些用于控制测试的编译，另一些参数用于控制测试的执行

### 控制编译的参数

1）-args

-args 指示 go test 把-args后面的参数带到测试中去，具体测测试函数会根据此参数来控制测试流程

-args后面可以附带多个参数，所有参数都将以字符串形式传入，每个参数作为一个string，并存放到字符串切片当中。

```go
func TestArgs(t *testing.T) {
    if !flag.Parsed() {
        flag.Parse()
    }
    argList := flag.Args()
    for _, arg := range argList {
        if arg == "cloud" {
            t.Log("Running in cloud.")
        } else {
            t.Log("Running in other mode")
        }
    }
}
```

 执行时带入参数：

<img width="506" alt="Image" src="https://github.com/user-attachments/assets/b1738d2b-22e1-4cfb-9120-9fab40df48e5" />

2)-json

-json用于指示go test将结果转换成json格式，以便在自动化测试解析时使用

<img width="1047" alt="Image" src="https://github.com/user-attachments/assets/a54c1683-142d-4539-a77d-e7122c307bfd" />

3）-o

-o指定生成的二进制可执行程序，并执行测试，测试结束不会删除该程序

<img width="538" alt="Image" src="https://github.com/user-attachments/assets/44f75367-511e-4b9a-82e0-a5c8dea80d11" />

### 控制测试的参数

1）-bench regexp

go test默认不执行性能测试，使用-bench才可以运行，而且只运行性能测试函数

其中的正则表达式用于筛选索要执行的性能测试。如果要执行所有的性能测试，则可以使用参数-baneh . 或-bench=.

比如有如下三个性能测试：

- func BenchmarkMakeSliceWithoutAlloc(b *testing.B)

- func BenchmarkMakeSliceWithAlloc(b *testing.B)

- func BenchmarkSetBytes(b *testing.B)

使用参数-bench=Slice，因为前两个测试因为包含Slice，所以前两个执行

2）-benchtime s

-benchtime 指定每个测试的执行时间，如果不指定，则使用默认时间1s

如：每个性能测试执行两秒 go test -bench -benchtime 2s

3）-cpu 1，2，4

-cpu 提供了一个cpu个数俩表，提供此列表后，测试按照这个列表指定的cpu数设置GOMAXPROCS并分别进行测试

3）-count n

-count 指定每个测试执行的次数，默认执行一次

例如：

<img width="554" alt="Image" src="https://github.com/user-attachments/assets/2d20416b-f28f-44e5-9aef-6de3651be67f" />

5）-failfast

默认情况下，go test 将执行所有匹配的测试i，并最后打印测试结果，无论成功或失败

-failfast 指定如果测试出现失败，则立即停止测试。在有大量的测试需要执行时，这样能够更快的发现问题

6）-list regexp

-list 只是列出匹配成功的测试函数，并不真正执行，而且不会列出子函数

7）-parallel n

-parallel n指定测试的最大并发数。

当使用 t.Parallel()方法时将测试转为并发时，将受到最大并发数的限制，默认情况下最多有

GOMAXPROCS个测试并发，其他的测试只能阻塞等待

8）-run regexp

-run regexp 的作用时根据正则表达式执行单元测试和示例测试，和-bench一样也是包含关系，并不是真正的正则

9）-timeout d

默认情况下，测试执行超过10分钟就会因超时自动退出

超时可以按秒，分，小时进行设置

 -timeout xs（xm|xh）或-timeout=xs（xm|xh）

10）-v

默认情况下，只打印简的测试结果，使用参数-v可以打印详细的日志

性能测试下，总是打印日志，因为日志又是会影响性能结果

11）-benchmem

默认情况下，性能测试只打印运行次数、每个操作耗时。使用-bencheme则可以打印每个操作分配的字节数和每个操作分配的对象数

<img width="848" alt="Image" src="https://github.com/user-attachments/assets/af798a9a-b748-4551-ac50-041bfd8fd9c6" />

## benchstat

分析benchmark的结果

官方推荐的性能测试分析工具benchstat

可使用 go get golang.org/x/perf/cmd/benchstat 命令快捷安装benchstat

> benchstat [options] old.txt [new.txt] [more.txt...]

<img width="794" alt="Image" src="https://github.com/user-attachments/assets/a1cdef7f-5234-4817-b0cd-5d5dcda08e63" />
