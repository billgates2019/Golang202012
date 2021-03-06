## 10.反射机制

### 10.1 反射是什么

> 反射的概念是由Smith在1982年首次提出的，主要是指程序可以访问、检测和修改它本身状态或行为的一种能力。

> Go 语言提供了一种机制在运行时更新变量和检查它们的值、调用它们的方法，但是在编译时并不知道这些变量的具体类型，这称为反射机制。

### 10.2 反射的作用

**1.在编写不定传参类型函数的时候，或传入类型过多时**

典型应用是对象关系映射

```go
type User struct {
  gorm.Model
  Name         string
  Age          sql.NullInt64
  Birthday     *time.Time
  Email        string  `gorm:"type:varchar(100);unique_index"`
  Role         string  `gorm:"size:255"` // set field size to 255
  MemberNumber *string `gorm:"unique;not null"` // set member number to unique and not null
  Num          int     `gorm:"AUTO_INCREMENT"` // set num to auto incrementable
  Address      string  `gorm:"index:addr"` // create index with name `addr` for address
  IgnoreMe     int     `gorm:"-"` // ignore this field
}

var users []User
db.Find(&users)
```


**2.不确定调用哪个函数，需要根据某些条件来动态执行**

```go
func bridge(funcPtr interface{}, args ...interface{})
```


第一个参数funcPtr以接口的形式传入函数指针，函数参数args以可变参数的形式传入，bridge函数中可以用反射来动态执行funcPtr函数。

### 10.3 反射的实现

Go的反射基础是接口和类型系统，Go的反射机制是通过接口来进行的。

Go 语言在 reflect 包里定义了各种类型，实现了反射的各种函数，通过它们可以在运行时检测类型的信息、改变类型的值。

#### 10.3.1 反射三定律

**1.反射可以将“接口类型变量”转换为“反射类型对象”。**

反射提供一种机制，允许程序在运行时访问接口内的数据。首先介绍一下reflect包里的两个方法**reflect.Value**和**reflect.Type**

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var Num float64 = 3.14

	v := reflect.ValueOf(Num)
	t := reflect.TypeOf(Num)

	fmt.Println("Reflect : Num.Value = ", v)
	fmt.Println("Reflect : Num.Type  = ", t)
}
```


返回

```go
Reflect : Num.Value =  3.14
Reflect : Num.Type  =  float64
```


上面的例子通过reflect.ValueOf和reflect.TypeOf将接口类型变量分别转换为反射类型对象v和t，v是Num的值，t也是Num的类型。

先来看一下reflect.ValueOf和reflect.TypeOf的函数签名

```go
func TypeOf(i interface{}) Type
```


```go
func (v Value) Interface() (i interface{})
```


两个方法的参数类型都是空接口

在整个过程中，当我们调用reflect.TypeOf(x)的时候，

当我们调用reflect.TypeOf(x)的时候，Num会被存储在这个空接口中，然后reflect.TypeOf再对空接口进行拆解，将接口类型变量转换为反射类型变量

**2.反射可以将“反射类型对象”转换为“接口类型变量”。**

定律2是定律1的反过程。

根据一个 reflect.Value 类型的变量，我们可以使用 Interface 方法恢复其接口类型的值。

```go
package main
import (
    "fmt"
    "reflect"
)
func main() {
    var Num = 3.14
    v := reflect.ValueOf(Num)
    t := reflect.TypeOf(Num)
    fmt.Println(v)
    fmt.Println(t)

    origin := v.Interface().(float64)
    fmt.Println(origin)
}
```


返回

```
3.14
float64
3.14
```


**3.如果要修改“反射类型对象”，其值必须是“可写的”。**

运行一下程序：

```go
package main
import (
    "reflect"
)
func main() {
        var Num float64 = 3.14
        v := reflect.ValueOf(Num)
        v.SetFloat(6.18)
}
```


出现panic了

```
panic: reflect: reflect.Value.SetFloat using unaddressable value

goroutine 1 [running]:
reflect.flag.mustBeAssignableSlow(0x8e)
	/usr/local/go/src/reflect/value.go:259 +0x138
reflect.flag.mustBeAssignable(...)
	/usr/local/go/src/reflect/value.go:246
reflect.Value.SetFloat(0x488ec0, 0xc00001a0b8, 0x8e, 0x4018b851eb851eb8)
	/usr/local/go/src/reflect/value.go:1609 +0x37
main.main()
	/home/ricardo/error.go:8 +0xb3
exit status 2
```


因为反射对象v包含的是副本值，所以无法修改。

我们可以通过CanSet函数来判断反射对象是否可以修改，如下：

```go
package main
import (
    "fmt"
    "reflect"
)
func main() {
    var Num float64 = 3.14
    v := reflect.ValueOf(Num)
    fmt.Println("v的可写性:", v.CanSet())
}
```


**小结**

1.反射对象包含了接口变量中存储的值以及类型。

2.如果反射对象中包含的值是原始值，那么可以通过反射对象修改原始值；

3.如果反射对象中包含的值不是原始值（反射对象包含的是副本值或指向原始值的地址），则该反射对象不可以修改。

### 10.4 反射的实践

**1.通过反射修改内容**

```go
var f float64 = 3.41
fmt.Println(f)
p := reflect.ValueOf(&f)
v := p.Elem()
v.SetFloat(6.18)
fmt.Println(f)
```


reflect.Elem() 方法获取这个指针指向的元素类型。这个获取过程被称为取元素，等效于对指针类型变量做了一个*操作

**2.通过反射调用方法**

```go
package main

import (
        "fmt"
        "reflect"
)

func hello() {
  fmt.Println("Hello world!")
}

func main() {
  hl := hello
  fv := reflect.ValueOf(hl)
  fv.Call(nil)
}

```


反射会使得代码执行效率较慢，原因有

1.涉及到内存分配以及后续的垃圾回收

2.reflect实现里面有大量的枚举，也就是for循环，比如类型之类的
