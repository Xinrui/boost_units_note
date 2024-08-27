# Boost.Units学习笔记
---
## 介绍
- `Boost.Units`是一个量纲分析库。由于使用模板元编程，其没有运行时开销，这一点有助于写出高性能的代码。  
- `Boost.Units`库作为单位转换的细粒度通用工具，支持任意的单位制模型（unit system model）和任意的值类型（value type）。
- `Boost.Units`提供了SI和CGS两种单位制；
提供了角度值、弧度制、百分度制、周制（revolution）四种平面角计量单位；
提供了开尔文、摄氏、华氏三种温标。
- 库的架构考虑了灵活性和可扩展性，添加新的单位和单位转换都很方便。

## 术语
- **基本量纲**（Base dimension）：直观来说，基本量纲表现了物理量的基本性质和特征；
在传统的量纲分析中，基本量纲包括长度（[L]），质量（[M]），时间（[T]）等等，但是我们可以使用任意的基本量纲，没有限制。
本质上来说，基本量纲是一种标签（tag）类型，不提供量纲分析功能。
- **量纲**（Dimension）：若干个基本量纲的集合，每个基本量纲都可能有不同的有理数幂。
如：长度=[L]^1，面积=[L]^2，速度=[L]^1/[T]^1，能量=[M]^1[L]^2/[T]^2都是量纲。
- **基本单位**（Base unit）：基本单位表示量纲的特定度量。例如，*长度*是距离的抽象度量，而*米*是距离的具体的基本单位。
基本单位是一种仅用于定义单位的标签类型，不支持量纲分析的运算，这一点与基本量纲非常相似。
- **单位**（Unit）：一组带有有理数幂的基本单位的集合，如[m]^1，[kg]^1，[m]^1/[s]^2。
- **单位制**（System）：对于我们感兴趣的特定问题来说，单位制是一组基本单位的集合，这组单位能够代表所有的可测量实体。
例如，SI 单位系统定义了七个基本单位：长度（[L]）（以米为单位）、质量（[M]）（以千克为单位）、
时间（[T]）（以秒为单位）、电流（[I]）（以安培为单位）、温度（[theta]）（以开尔文为单位）、
数量（[N]）（以摩尔为单位）和发光强度（[J]）（以坎德拉为单位）。
SI系统内的所有可测量实体都可以表示为这七个基本单位的各种整数或有理数幂的乘积。
- **量**（Quantity）：量表示单位的具体数量。因此，尽管米是国际单位制中长度的基本单位，
5.5米才是国际单位制中的长度量。

## `example/tutorial.cpp`
这个例子介绍了国际单位的使用，代码的意图非常明确，不需要更多的解释：
```
#include <complex>
#include <iostream>

#include <boost/typeof/std/complex.hpp>

#include <boost/units/systems/si/energy.hpp>
#include <boost/units/systems/si/force.hpp>
#include <boost/units/systems/si/length.hpp>
#include <boost/units/systems/si/electric_potential.hpp>
#include <boost/units/systems/si/current.hpp>
#include <boost/units/systems/si/resistance.hpp>
#include <boost/units/systems/si/io.hpp>

using namespace boost::units;
using namespace boost::units::si;

constexpr
quantity<energy>
work(const quantity<force>& F, const quantity<length>& dx)
{
    return F * dx; // Defines the relation: work = force * distance.
}

int main()
{
    /// Test calculation of work.
    quantity<force>     F(2.0 * newton); // Define a quantity of force.
    quantity<length>    dx(2.0 * meter); // and a distance,
    quantity<energy>    E(work(F,dx));  // and calculate the work done.

    std::cout << "F  = " << F << std::endl
              << "dx = " << dx << std::endl
              << "E  = " << E << std::endl
              << std::endl;

    /// Test and check complex quantities.
    typedef std::complex<double> complex_type; // double real and imaginary parts.

    // Define some complex electrical quantities.
    quantity<electric_potential, complex_type> v = complex_type(12.5, 0.0) * volts;
    quantity<current, complex_type>            i = complex_type(3.0, 4.0) * amperes;
    quantity<resistance, complex_type>         z = complex_type(1.5, -2.0) * ohms;

    std::cout << "V   = " << v << std::endl
              << "I   = " << i << std::endl
              << "Z   = " << z << std::endl
              // Calculate from Ohm's law voltage = current * resistance.
              << "I * Z = " << i * z << std::endl
              // Check defined V is equal to calculated.
              << "I * Z == V? " << std::boolalpha << (i * z == v) << std::endl
              << std::endl;
    return 0;
}
```
输出：
```
F  = 2 N
dx = 2 m
E  = 4 J

V   = (12.5,0) V
I   = (3,4) A
Z   = (1.5,-2) Ohm
I*Z = (12.5,0) V
I*Z == V? true
```

## 量纲分析

### 形式化
...

### 实现
在实现上说，基本量纲是一种标签类型，由复合量纲是类型的列表。
因此，为了能把任意的复合量纲转化为基本量纲的组合，我们需要提供一种对基本量纲排序的机制。
`Boost.Units`库给每个基本量纲都指定了一个唯一的整数。
`base_dimension`类（定义在`boost/units/base_dimension.hpp`）
使用了[CRTP](https://zh.cppreference.com/w/cpp/language/crtp)[^1]
技术来保证对基本量纲指定的序数是唯一的。

```
template<class Derived, long N> struct base_dimension { ... };
```

基于此，我们可以定义长度、质量、时间的基本量纲：
```
/// base dimension of length
struct length_base_dimension : base_dimension<length_base_dimension,1> { };
/// base dimension of mass
struct mass_base_dimension : base_dimension<mass_base_dimension,2> { };
/// base dimension of time
struct time_base_dimension : base_dimension<time_base_dimension,3> { };
```

序数的选择是任意的，只要每个标签的枚举值是唯一的就好：
- 不唯一的序数会导致编译错误
- 负的序数被`Boost.Units`库保留

要定义复合量纲，我们只需通过使用`dim`类来封装基本量纲和指数上的`static_rational`，来创建符合`Boost.MPL`的基本量纲类型列表即可。
`make_dimension_list`作为包装器以确保生成的类型由基本量纲组成：

```
typedef make_dimension_list<
    boost::mpl::list< dim< length_base_dimension,static_rational<1> > >
>::type   length_dimension;

typedef make_dimension_list<
    boost::mpl::list< dim< mass_base_dimension,static_rational<1> > >
>::type     mass_dimension;

typedef make_dimension_list<
    boost::mpl::list< dim< time_base_dimension,static_rational<1> > >
>::type     time_dimension;
```

使用`base_dimension`提供的`typedef`会更容易：
```
typedef length_base_dimension::dimension_type    length_dimension;
typedef mass_base_dimension::dimension_type      mass_dimension;
typedef time_base_dimension::dimension_type      time_dimension;
```
上面两种写法是完全等价的。类似地，使用一个类型列表定义复合类型：
```
typedef make_dimension_list<
    boost::mpl::list< dim< length_base_dimension,static_rational<2> > >
>::type   area_dimension;

typedef make_dimension_list<
    boost::mpl::list< dim< mass_base_dimension,static_rational<1> >,
                      dim< length_base_dimension,static_rational<2> >,
                      dim< time_base_dimension,static_rational<-2> > >
>::type    energy_dimension;
```

`derived_dimension`可以简化这一过程：
```
typedef derived_dimension<length_base_dimension,2>::type  area_dimension;
typedef derived_dimension<mass_base_dimension,1,
                          length_base_dimension,2,
                          time_base_dimension,-2>::type   energy_dimension;
```

[^1]:奇特重现模板模式（curiously recurring template pattern）

## 单位
我们定义，单位是一组有着有理数幂的基本单位之集合。因此，力的量纲的国际单位是[kg·m·s^-2]。
我们用单位制这一概念，来指定从量纲到一个特定单位的映射（如SI）。
这样我们就可以只关心量纲，而非基本维度。

单位和量纲一样，都是没有关联值的编译期常量，都遵循相同的代数运算。单位制的存在，
是为了确保有相同量纲的单位（比如英尺和米）不会在计算中被无意之间搞混。

可以想到，有以下两种不同的单位制:

1. **齐次单位制**：基本单位线性无关。例如，SI单位制有七个基本量纲和七个相应的基本单位。
2. **非均匀单位制**：“非均匀”这一术语，指的是存储每个相关基本单位的指数。
一些单位只能以这种方法表示，例如，单位为m·ft的面积是非均匀的，因为米和英尺有相同的量纲。
因此，仅存储一个维度和一组基本单位并不能产生唯一的解决方案。
另一个例子是航空中使用的一个经验方程：H = (r/C)^2，
其中H是雷达波束高度（单位为英尺），r是雷达范围（单位为海里）。为了保证方程量纲正确，
常数C的单位必须是海里/根号（英尺），这样混合了两种不同的基本长度单位。

单位概念由定义在`boost/units/unit.hpp`的模板类`unit`实现：
```
template<class Dim,class System> class unit;
```
除了支持编译期的量纲分析操作，`unit`类也支持运行时的`+, -, *, /`运算符。
由于量纲的幂(包括根号运算)必须要在编译期完成计算，我们没有办法提供`unit`类的`std::pow`重载。
这些操作通过非成员函数`pow`和`root`支持，它们的模板参数可以是`int`或者`static_rational`，
返回的类型是在两个工具类（`power_typeof_helper`和`root_typeof_helper`）中定义的类型。

### 基本单位
基本单位的定义和基本量纲相似：
```
template<class Derived, class Dimensions, long N> struct base_unit { ... };
```
同样地，负序数被保留。
作为示例，根据上面给出的基本量纲，我们将在下文中实现SI单位制的一个子集，
演示完整功能系统所需的所有步骤。
首先，我们简单定义一个单位系统，其中包括常用单位的类型定义：

```
struct meter_base_unit : base_unit<meter_base_unit, length_dimension, 1> { };
struct kilogram_base_unit : base_unit<kilogram_base_unit, mass_dimension, 2> { };
struct second_base_unit : base_unit<second_base_unit, time_dimension, 3> { };

typedef make_system<
    meter_base_unit,
    kilogram_base_unit,
    second_base_unit>::type mks_system;

/// unit typedefs
typedef unit<dimensionless_type,mks_system>      dimensionless;

typedef unit<length_dimension,mks_system>        length;
typedef unit<mass_dimension,mks_system>          mass;
typedef unit<time_dimension,mks_system>          time;

typedef unit<area_dimension,mks_system>          area;
typedef unit<energy_dimension,mks_system>        energy;
```
在`boost/units/static_constant.hpp`提供的宏`BOOST_UNITS_STATIC_CONSTANT`
可以方便我们在头文件中定义ODR和线程安全的常量。
据此，我们可以为支持的单位定义一些常量，来简化变量定义：
```
/// unit constants 
BOOST_UNITS_STATIC_CONSTANT(meter,length);
BOOST_UNITS_STATIC_CONSTANT(meters,length);
BOOST_UNITS_STATIC_CONSTANT(kilogram,mass);
BOOST_UNITS_STATIC_CONSTANT(kilograms,mass);
BOOST_UNITS_STATIC_CONSTANT(second,time);
BOOST_UNITS_STATIC_CONSTANT(seconds,time);

BOOST_UNITS_STATIC_CONSTANT(square_meter,area);
BOOST_UNITS_STATIC_CONSTANT(square_meters,area);
BOOST_UNITS_STATIC_CONSTANT(joule,energy);
BOOST_UNITS_STATIC_CONSTANT(joules,energy);
```
如果需要单位的文本输出，我们可以给每个基本量纲[^2]标签都特化一个`base_unit_info`类：

```
template<> struct base_unit_info<test::meter_base_unit>
{
    static std::string name()               { return "meter"; }
    static std::string symbol()             { return "m"; }
};
```

对于`kilogram_base_unit`和`second_base_unit`也是类似的。
未来，`Boost.Units`将通过`locale`本地化库实现国际化，来提供一个更加灵活的系统。
`base_unit_info`类的`name()`和`symbol()`方法提供了基本单位的全名和缩写。
有了这些定义，我们就有了量纲系统的雏形，这个系统可以确定任意单位计算的简化后的量纲。

[^2]:原文如此。这里应该是基本单位，而非基本量纲

### 缩放基本单位
我们可以将一个基本单位定义为另一个基本单位的倍数。比如，库中是这样定义`kilogram_base_unit`的：
```
struct gram_base_unit : boost::units::base_unit<gram_base_unit, mass_dimension, 1> {};
typedef scaled_base_unit<gram_base_unit, scale<10, static_rational<3> > > kilogram_base_unit;
```
这样就定义了1千克是10^3克。
这样做有几个好处：
- 相比于把kg视为独立单位，这样做更能反映kg的真正的含义。
- 如果定义了克/千克和其他单位的直接转换，那么只需要一次特化，一切都会正常工作。
- 类似地，如果将克的符号定义为"g"，那么千克的符号将直接变为"kg"。

### 缩放单位
我们也可以缩放作为一个整体的`unit`，而不是缩放组成它的基本单位。为此，我们可以使用元函数
`make_scaled_unit`。这个功能的动机主要来源于定义在`boost/units/systems/si/prefixes.hpp`中的度量前缀。

下面是一个简单的例子：
```
typedef make_scaled_unit<si::time, scale<10, static_rational<-9> > >::type nanosecond;
```
纳秒是`unit`的一个特化，可以直接用在一个量里面。
```
quantity<nanosecond> t(1.0 * si::seconds);
std::cout << t << std::endl;    // prints 1e9 ns
```

## 量
**量**是一个值，其类型任意，具有一个特定的单位。例如，米是一个单位，而3.0米是一个量。
量遵循两种独立的代数计算：值类型的原生代数计算，以及相关单位的量纲分析代数计算。除此之外，
为了简化量的定义，单位和量之间也定义了代数操作；这实际上和具有单位值量的代数计算是一致的。

量由定义在`boost/units/quantity.hpp`的模板类`quantity`实现：
```
template<class Unit,class Y = double> class quantity;
```

单位类型`Unit`和值类型`Y`都是模板参数，如果没有指定，后者的类型是双精度浮点数。
值类型必须要有一个正常的拷贝构造函数和一个正常的赋值运算符。
标量和单位的运算，标量和量的运算，单位和量的运算，以及量和量的运算之间，都定义了
`+, -, *, /`的代数运算。除此之外，可以使用`pow<R>`和`root<R>`来计算整数/有理数的幂和根。
最后，库提供了一组标准的比较运算符`==, !=, <, <=, >, >=`比较来自相同单位系统的量。
如果量可以被比较的话，这些运算符都只是委托给了值类型的，相应的运算符。

### 非均匀运算符
对于大多数常见的值类型来说，算术运算符的返回值类型就是值类型本身。
比如，两个双精度浮点数之和还是一个双精度浮点数。
然而，这个规则不一定成立。一个简单的反例是有理数的运算：
- 两个自然数之和是自然数
- 两个自然数之差是整数
- 两个自然数之积是自然数
- 两个自然数之商是有理数

`Boost.Units`支持任意的值类型代数运算，这些运算包括加减乘除、有理数幂和开根号。
这些运算符的结果由`Boost.TypeOf`推导得到。支持`typeof`的编译器会自动推导出合适的值类型，
对于不支持`typeof`的编译器，需要手动注册所有用到的类型。在自然数这个例子中，这相当于以下代码：
```
BOOST_TYPEOF_REGISTER_TYPE(natural);
BOOST_TYPEOF_REGISTER_TYPE(integer);
BOOST_TYPEOF_REGISTER_TYPE(rational);
```

### 转换
转换只对量才有意义，因为这意味着存在至少一个乘法比例因子，可能还有仿射线性偏移[^3]。
简化单位之间相互转换的宏定义在`boost/units/conversion.hpp`和
`boost/units/absolute.hpp`（用于仿射变换）中。

宏`BOOST_UNITS_DEFINE_CONVERSION_FACTOR`定义了从第一个单位类型转换为第二个单位类型的比例因子。第一个参数必须是`base_unit`，
第二个参数可以是`base_unit`或者`unit`。

让我们声明一个简单的基本单位：
```
struct foot_base_unit : base_unit<foot_base_unit, length_dimension, 10> { };
```

现在，我们想把英尺转换到米，反之亦然。1英尺等于0.3048米，所以我们可以这样写：

```
BOOST_UNITS_DEFINE_CONVERSION_FACTOR(foot_base_unit, meter_base_unit, double, 0.3048);
```

也可以使用`SI length typedef`：
```
BOOST_UNITS_DEFINE_CONVERSION_FACTOR(foot_base_unit, SI::length, double, 0.3048);
```

由于长度的国际单位是米，这两个定义是等价的。如果定义了这些转换，这些单位，
及其缩放形式的转换会自动进行。

如果无法直接转换，宏`BOOST_UNITS_DEFAULT_CONVERSION`指定了到基本单位的转换，
只要一个简单的特化：
```
struct my_unit_tag : boost::units::base_unit<my_unit_tag, boost::units::force_type, 1> {};
// define the conversion factor
BOOST_UNITS_DEFINE_CONVERSION_FACTOR(my_unit_tag, SI::force, double, 3.14159265358979323846);
// make conversion to SI the default.
BOOST_UNITS_DEFAULT_CONVERSION(my_unit_tag, SI::force);
```

[^3]: 译者注：比如华氏温标和摄氏温标的换算：°F = 1.8(°C) + 32

### 量的构造和转换
库的设计要求，在操作带量纲的量时，安全性高于便利性。具体而言，在构造量的时候需要指定值和单位。
禁止从标量类型直接构造（尽管在必要的情况下，可以使用静态成员函数`from_value`来启用这个功能）。
此外，对引用使用`quantity_cast`可以直接取到`quantity`变量的底层值。库提供了显式构造函数，
以启用在不同单位制下，量纲一致的量的转换。单位制的隐式转换只在简化后的量纲一致的情况下才会进行，
比如等价的单位在不同系统的任意转换（SI的秒和CGS的秒），同时在编译时检查（可能是无意的）单位制不匹配。
赋值遵循相同的原则，但是有一个例外：当单位退化到无量纲的情况下，允许通过类模板特化进行底层值的隐式转换。
不同值类型的量只有在值类型本身可隐式转换时才能隐式转换。`quantity`类也定义了一个`value()`函数来直接取到底层值。

总而言之，在以下条件下允许转换：
- 如果`Y`和`Z`可以隐式转换，允许`quantity<Unit,Y>`和`quantity<Unit,Z>`的隐式转换
- 如果`Y`和`Z`可以隐式转换，允许`quantity<Unit,Y>`和`quantity<Unit,Z>`之间的赋值
- 如果`Unit1`和`Unit2`有相同的量纲，且`Y`和`Z`可以隐式转换，允许`quantity<Unit1,Y>`和`quantity<Unit2,Z>`的显式转换
- 如果`Unit1`和`Unit2`有完全相同的基本量纲组合，且`Y`和`Z`可以隐式转换，
允许`quantity<Unit1,Y>`和`quantity<Unit2,Z>`的隐式转换
- 在和隐式转换相同的条件下，允许`quantity<Unit1,Y>`和`quantity<Unit2,Z>`之间的赋值
- 可以使用静态成员函数`from_value`和一个有着类型`Y`的值构造一个`quantity<Unit,Y>`。这么做自然
会绕过对新分配的值的类型检查，应该在必要的时候[^4]才用这个方法。

只要允许隐式转换，显式转换自然也是合法的。

无量纲的量没有单位，它们表现得就像普通的标量一样，并且可以和底层值类型（以及和底层值类型相容的类型）进行隐式转换。

[^4]: 比如在使用经验公式计算某些物理量的时候

## 例子
...

## FAQ
...