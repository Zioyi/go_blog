# 常量
Pob Pike

2014年8月24日

[原文](https://blog.golang.org/constants)

## 介绍
Go是一门静态语言，它不允许不同数字类型间的操作。你不能将一个浮点数（float64）和一个整数（int)相加，也不能讲一个32位整数（int32）和一个通用整数（int)相加。这些写法也是非法的：1e6*time.Second、math.Expr(1)、1<<('\t'+2.0)。在Go中，常量和变量不同，它很像是常数（regular number)。这篇文章将解释其中缘由。

## 背景：C
在之前关于Go的思考中，我们讨论过C语言和C延伸出的语言允许你混合不同数字类型会引发很多问题。许多匪夷所思的bug，异常中断和兼容性问题都是由于表达式里由不同长度的整型和符号类型（signedness）组成所导致的。即便是对经验丰富的C语言工程师，对于下面代码的计算结果也会迟疑。
```
unsigned int u = 1e9;
long singed int i = -1;
... i + u ...
```
最后的答案是多少？它是什么类型的？有符号还是无符号的？

代码里还潜伏这bug。

C语言有一系列“通用算数转换”规则，it is an indicator of their subtlety that they have changed over the years (introducing yet more bugs, retroactively).

当设计Go时，我们决定避免这个雷区，我们不允许不同数字类型之间的混合操作。如果你想将i和u相加，你必须显示声明你最终想要得到的类型。
```
var u uint
var i int
```
你可以写 uint(i)+u 或者 i+int(u)，这样清楚地表达了相加操作的结果类型，你不能像C语言那样写 i+u。即使int是32bit位的，你也不能将int类型数字和int32类型数字混合操作。

这个强制要求消除类一些通常的bug和异常，他是Go中重要的属性。但它也需要代价：有时需要程序员去用笨拙的数字类型装换去装饰代码以清楚地表达出含义。

那常量怎么办？在上面的声明里，怎么合法地写 i=0 或者 u=0？0的类型是什么？
```
var i int = int(0)
```
这种声明方式显然是不合理的。

我们很快意识到答案就是让数字常量的工作方式和其他C类语言不同。在更多的思考和实验之后，我们想出了我们相信最正确的设计，将程序员从总是要转换常量中解放出来，他们可以这样写：math.Sqrt(2)，编译器也不会报错。

总之，在Go中，常量总是行得通的。让我们看看发生了什么。

## 术语
首先在Go中，const是一个关键字，用一个标量（比如：2、3.14159、"scrumptious"）来声明一个名字。这样的值（命名或其他形式）在Go中称为常量。常量也可通过由常量构建的表达式创建，比如：2+3、2+3i、Pi/2、("go" + "pher")。

某些语言没有常量，而另一些语言则具有常量的更一般定义或const单词的应用。例如，在C和C++中，const是一个类型限定符，可以将更多复杂值的更复杂属性进行编码。

但是在Go中，常量仅仅是单一不可变的值，现在开始我们只讨论Go中的常量。

## 字符串
这里有很多种类型的数字常量：整型、浮点型、rune、有符号型、无符号型、虚数型、复数型。让我们以简单的字符串常量作为开始。字符串常量很容易理解，可以在其中探索Go中常量的类型问题。

一个字符串常量是双引号闭合的文本（Go也支持原生字符串写法，即用反引号闭合``````）。下面是一个字符串常量：
```
"Hello, 世界"
```

字符串常量的类型是什么？string是错误的答案。

它是无类型字符串常量，即它没有固定的类型。没错，它是一个字符串，但它的类型不是Go里面的string。即便提供一个变量名，它仍然是无类型字符串常量：
```
const hello = "Hello, 世界"
```

这样声明之后，hello还是一个无类型字符串常量，一个无类型常量只有一个没有定义类型的值，所以它不受强制相同类型才可操作的限制。

正是无类型常量的概念让我们可以在Go中自由使用常量。

那么，有类型字符串常量是什么？下面是一个有类型的例子：
```
const typedHello string = "Hello, 世界"
```

注意typedHello的声明，有一个显式string类型在等号前面。这意味着typedHell是string类型，它不可以被分配给其他类型。正确示例：
```
var s string
s = typeHello
fmt.Println(s)
```
错误示例：
```
type MyString string
var m Mystring
m = typedHello // Type error
fmt.Println(m)
```
变量m是Mystring类型的，它不能被其他类型值赋值。它只能被MyString类型值赋值，像这样：
```
const myStringHello MyString = "Hello, 世界"
m = myStringHello // OK
fmt.Println(m)
```
或者通过强制类型装换来解决：
```
m = MyString(typedHello)
fmt.Println(m)
```

回到我们的无类型字符串常量，因为它没有类型，所有把它赋值给一个有类型的变量不会引起类型错误。我们可以这样写：
```
m = "Hello, 世界"
```
或者
```
m = hello
```

不像有类型常量typedHello和myStringHello，无类型常量"Hello, 世界"和hello没有类型，所有把他们赋值给任何类型兼容string的变量都不会出错。

这些无类型字符串常量当然是字符串，所以他们只能用在string允许的地方，但他们没有类型。

## 默认类型
做为一名Go程序员，你肯定见过这里的声明：
```
str := "Hello, 世界"
```

现在，你可能会问：“如果常量是无类型的，那比变量str是什么类型的？”答案是string，无类型常量会有一个默认类型，会在需要时自动将自己转换为该类型。对于无类型字符串常量，默认类型是string，所以
```
str := "Hello, 世界"
```
或者
```
var str = "Hello, 世界"
```
意思都是
```
var str string = "Hello, 世界"
```

你可以把无类型常量当成一种在特殊空间的值，它们不会受到Go类型系统的大部分限制。但需要将它赋值给有类型的变量时，它们把自己的默认类型告诉变量。在这个例子中，str变成了一个string类型的值，因为无类型字符串常量提供它们的默认类型string用来声明。

在上面的声明中，一个变量会被声明并带着类型和初始值。有时当我们使用常量时，并没有明确的目标值，例如：
```
fmt.Printf("%s", "Hello, 世界")
```
输出：
```
Hello, 世界
```

fmt.Printf的签名是
```
func Printf(format string, a ...interface{}) (n int, err error)
```
format之后的参数是接口类型（interface）。但调用fmt.Printf时，使用一个无类型常量作为参数传递时，参数的具体类型是常量的默认类型。这个过程类似于我们之前使用无类型的字符串常量声明初始化值时看到的过程。

你可以看到洗下面例子的结果，使用%v打印值，%T打印值的类型：
```
fmt.Printf("%T: %v\n", "Hello, 世界", "Hello, 世界")
fmt.Printf("%T: %v\n", hello, hello)
```
输出：
```
string: Hello, 世界
string: Hello, 世界
```

如果常量有类型，那么会传递到接口，像下面这样：
```
fmt.Printf("%T: %v\n", myStringHello, myStringHello)
```
输出：
```
main.MyString: Hello, 世界
```

总结一下，一个有类型常量会遵循Go里所有的类型规则；一个无类型常量可以不用遵守，并且可以更自由地混合和匹配。无类型常量的默认类型只会在必要时且没有其他类型信息时被使用。

## 语法决定默认类型
一个无类型常量的默认类型被它的语法所决定。对于字符串常量，默认类型只会是string。对于数字常量，默认类型有多种。整型常量默认是int，浮点型常量默认是float64，rune类型常量默认是rune(int32)，虚数类型常量默认是comlex128。例子：
```
fmt.Printf("%T %v\n", 0, 0)
fmt.Printf("%T %v\n", 0.0, 0.0)
fmt.Printf("%T %v\n", 'x', 'x')
fmt.Printf("%T %v\n", 0i, 0i)
```
输出：
```
int 0
float64 0
int32 120
complex128 (0+0i)
```

## Booleans
无类型boolean常量和无类型字符串常量是一样的。true和false是无类型boolean常量，他们可以被赋值给任何boolean变量，一旦确定了类型，就不能和boolean变量混合操作了：
```
type MyBool bool
const True = true
const TypedTrue bool = true
var m Mybool
mb = true       // OK
mb = True       // OK
mb = TypedTrue  // Bad
fmt.Println(mb)
```

## 浮点类型
浮点常量和boolean常量有很多相同的地方：
```
type MyFloat64 float64
const Zero = 0.0    // 无类型浮点常量
const TypedZero float64 = 0.0
var mf MyFloat64
mf = 0.0            // OK
mf = Zero           // OK
mf = TypedZero      // Bad
fmt.Println(mf)
```
唯一不同的地方是Go里面有两种浮点类型：float32和float64。浮点常量默认的类型是float64，但是可以把无类型浮点常量赋值给float32类型的变量：
```
var f32 float32
f32 = 0.0
f32 = Zero      // OK: Zero is untyped
f32 = TypedZero // Bad: TypedZero is float64 not float32.
fmt.Println(f32)
```

我们来用浮点数介绍一下溢出的问题。

数字常量存储在一个任意精度的数字空间了，它们是常规的数字。但当他们被分配给一个变量时，大小必须在变量的类型所支持的范围内。我们可以给一个常量声明一个非常大的值：
```
const Huge = 1e1000
```

虽然我们声明了Huge这个常量，但是我们不能把Huge分配给其他变量，甚至无法打印出来。下面语句甚至不会编译：
```
fmt.Println(Huge)
```

会报错："constant 1.00000e+1000 overflows float64"。但是Huge是可用的：我们可以在表达式里用它和其他常量进行运算，如果结果在存在float64的范围里，如下：
```
fmt.Println(Huge / 1e999)
```
输出：
```
10
```

另外，浮点常量会有更高的精读，所以用他们计算更加准确。在math包中定义的常量比float64类型的变量拥有更多的精度位。下面是math.Pi的定义：
```
Pi = 3.14159265358979323846264338327950288419716939937510582097494459
```

当Pi被分配给一个变量时，会有精读丢失；该变量会是float64或float32类型来尽可能接近原来的值。比如：
```
pi := math.Pi
fmt.Println(pi)
```
输出：
```
3.141592653589793
```

数字常量拥有更多的精度位，意味着在进入比如：Pi/2这样的计算中结果的精度很高，直到结果被分配给变量时精度才会丢失。并且在计算过程不会出现无穷大、溢出和NaN的问题。（如果表达式中有除以常量0的操作，在编译时会报错。

## 复数
复数常量也和浮点数常量类似。
```
type MyComplex128 complex128
const I = (0.0 + 1.0i)
const TypedI complex128 = (0.0 + 1.0i)
var mc MyComplex128
mc = (0.0 + 1.0)    // OK
mc = I              // Ok
mc = TypedI         // Bad
fmt.Println(mc)
```

复数常量的默认类型是complex128, 是由两个float64的值组成，有很大的精度。

声明一下，在例子中，我们写了完整的表达式(0.0+1.0i)，其实我们可以简写成0.0+1.0i、1.0i或者1i。

我们知道在Go中，数字常量只是一个数字，如果一个复数没有虚数部分，它会是什么？实数吗？
```
const Two = 2.0 + 0i
```
Two数一个无类型复数常量，即使它没有虚数部分，表达式的语法定义了它的默认类型是complex128。因此，如果我们用它去声明一个变量，默认类型将会是complex128。
```
s := Two
fmt.Printf("%T: %v\n", s, s)
```
输出：
```
complex128: (2+0i)
```

在数值上，Two既可以被存储在float32空间里，也可以存储在float64空间里，不会发生精度丢失。我们在初始化或分配时把Two分配给float64是没问题的：
```
var f float64
var g float64 = Two
f = Two
fmt.Printf(f, "and", g)
```
输出：
```
2 and 2
```
即使Two是复数常量，也可以将其分配给浮点类型变量。这种跨界操作是很有用的。

## 整型
最后我们来到了整型。他们的种类更多，不同的大小、有符号或无符号等等，他们遵循一样的规则：
```
type MyInt int
const Three = 3
const TypedThree int = 3
var mi Myint
mi = 3          // OK
mi = Three      // OK
mi = TypedThree // Bad
fmt.Println(mi)
```
上面的类型对所有的整数类型都适用，整数类型有：
```
int int8 int16 int32 int64
uint uint8 uint16 uint32 uint64
uintptr
```
（还有uint8也叫byte, int32也叫rune）。整数类型有很多，但是在整数常量的工作方式都是相似的，现在我们来具体看一看。

如上所述，整型有两种形式，每种形式都有自己的默认类型：对于像123、0xFF、-14这些简单的常量，默认类型是int；像'a'、'世'、'\r'这些单引号单字符，默认类型是rune。

没有哪种常量形式的默认类型无符号整型。但是我们可以利用无类型常量的灵活性，初始化出无符号整型变量。这就和我们可以用虚部为0的复数初始化出float64变量一样。下面有几中初始化uint的方式，每种方法都是明确指定了类型为uint：
```
var u uint = 17
var u = uint(17)
u := uint(17)
```

整型也有类型浮点数那样数值范围的问题，不同整型之间的转换、分配可能会出现问题。可能会有两个问题发生：大范围的值转换为小范围的值；负数分配到无符号整型上。例如：int8的范围是-128到127，所以如果常量超出这个范围将不能被分配到一个int8类型的变量上：
```
var i8 int8 = 128 // Error: too large.
```
同样，uint8（或者叫byte）的范围是0到255，不在这个范围中的常量也不能被分配到一个uint8变量上：
```
var u8 uint8 = -1 // Error: negative value.
```
下面的代码也会被类型检查捕捉到错误：
```
type Char byte
var c Char = '世' // Error: '世' has value 0x4e16, too large
```
如果编译器检查出常量的用法错误，真实太悲伤了。

## 练习：最大的无符号整型
这里有一个有意思的小练习。我们如何用常量表示uint中的最大值呢？如果我们是说uint32，我们可以这个样写：
```
const MaxUint32 = 1<<32 - 1
```
但是我们要的是uint，而不是uint32。int和uint类型没有确定的bit位数，32位或者64位。因为具体的bit位数取决架构，我们不能写一个简单的值。

...省略...

最简单的uint值是有类型常量uint(0)，如果uint是32位，那么uint(32)就用32位0bit，反之uint是64位，那么uint(0)就有64位0bit。如果我们将这些bit位倒置成1，我们将会得到uint的最大值：
```
const MaxUint = ^uint(0)
fmt.Printf("%x\n", MaxUint)
```
无论当前环境uint32的bit位是多少，这个常量都正确表示uint变量能承载的最大值。

## 数字
在Go中无类型常量的概念表示所有的数字常量，无论整型、浮点型、复数型还是字符值，都存储在一种统一的空间里。我们将它们作用在变量、复制和运算中时，类型才会有意义。只要我们在数字常量的世界里，我们就可以随意混合不同的类型进行运算，下面所有的常量的数值都是1：
```
1
1.000
1e3-99.0*10-9
'\x01'
'\u0001'
'b' - 'a'
1.0+3i-3.0i
```
因此，即使他们有不同的隐含默认类型，写做无类型的常量可以被分配任何整型变量：
```
var f float32 = 1
var i int = 1.000
var u uint32 = 1e3 - 99.0*10.0 - 9
var c float64 = '\x01'
var p uintptr = '\u0001'
var r complex64 = 'b' - 'a'
var b byte = 1.0 + 3i - 3.0i

fmt.Println(f, i, u, c, p, r, b)
```
输出：
```
1 1 1 1 1 (1+0i) 1
```

你甚至可以这样写：
```
var f = 'a' * 1.5
fmt.Println(f)
```
输出：
```
145.5
```

尽管在Go中同一个表达式中混合使用浮点数和整数变量或者int和int32变量是不合法的，但是这种灵活性让下面的写法可行，即不同类型的数字常量是可以混合的：
```
sqrt2 := math.Sqrt(2)
```
或者
```
const millisecond = time.Second/1e3
```
或者
```
bigBufferWithHeader := make([]byte, 512+1e6
```
结果也是你期望的那样。

因为在Go里，数字常量的工作方式就像普通的数字一样。
