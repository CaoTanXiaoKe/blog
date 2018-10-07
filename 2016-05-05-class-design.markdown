---
layout:     post
title:      "一个简单类的简单设计"
subtitle:   "类的设计，类的实现，编程规范，读书笔记"
date:       2016-05-05
author:     "ChenWenKe"
header-img: "img/post-bg-class-design.jpg"
tags:
    - 设计
    - 读书
    - C++
    - 编程思想
---

> 数据抽象，继承和动态绑定构成了面向对象编程的基础. 

#### 设计 class 犹如设计 type

- 新 type 的对象应该如何被创建和销毁？
- 对象的初始化和对象的赋值该有什么样的差别？
- 新type的对象如果被 pasted-by-value （以值传递），意味着什么？
- 什么是新 type 的 “合法值” ？　　　
- 你的新 type 需要配合某个继承图系（inheritance graph）吗？
- 你的新 type 需要什么样的转换？
- 什么样的操作符和函数对此新 type 而言是合理的？
- 什么样的标准函数应该驳回？
- 谁该取用新 type 的成员？
- 什么是新 type 的“未申明接口”？
- 你的新 type 有多么一般化？

这里通过对一个简单类的设计和实现来说明一下面向对象编程的这三个思想。 

## 类的设计 
- 编写使用对象的场景（[用例](#用例)）。
- 类需要的成员函数（[接口](#接口)）包括外部函数。
- 确定类的[成员及其访问权限](#成员及其访问权限) 
- 类所需要的[所有函数](#所有函数)（包括初始化，移动，赋值，销毁）。
- [实现类](#实现类)（适当应用继承，多态，虚函数）
- 进行测试 ——> 使用 ——> 维护 ——> 重构

## Sales_data类
现在假定我们要做一个书店程序，需要定义一个Sales_data类。Sales_data类的**作用**是表示一本书的**总销售额**，**售出册数**和**平均售价**。开始时我们无需关心这些数据时如何存储的，如何进行计算的。我们不需要关心类的实现，只需要知道**如何使用这个类**，这个类的对象可以**执行什么操作**。 
- 调用一个名为 isbn 的函数从一个 Sales_data 对象中提取 ISBN 书号。 
- 用输入运算 (>>) 和输出运算符 (<<) 读，写 Sales_data 类型的对象。 
- 用赋值运算符 (=) 将一个 Sales_data 对象的值赋予另一个 Sales_data 对象。 
- 用加法运算符 (+) 将两个 Sales_data 对象相加。 两个对象必须表示同一本书（相同的ISBN）。加法结果是一个新的 Sales_data 对象，其 ISBN 与两个运算对象相同，而其总销售额和售出册数则是两个运算对象的对应值之和。 
- 使用复合运算符 (+=) 将一个 Sales_data 对象加到另一个对象上。 


### 用例
1.  **读写 Sales_data 用例**

```cpp
#include <iostream>
#include "Sales_data.h"
int main()
{
    Sales_data book;    
    // 读入 ISBN 号，售出的册数以及销售价格
    std::cin >> book; 
    // 写入 ISBN，售出的册数，总销售额和平均价格。 
    std::cout << book << std::endl; 
    return 0; 
}
```
如果输入: `0-201-70353-X 4 24.99` 则输出: `0-201-70353-X 4 99.96 24.99`


2.  **Sales_data 对象的加法用例**

```cpp
#include <iostream>
#include "Sales_data.h"
int main()
{
    Sales_data item1, item2;    
    std::cin >> item1 >> item2;     // 读取一对交易记录
    std::cout << item1 + item2 << std::endl; 
    return 0; 
} 
```
如果输入: <br/>
`0-201-70353-X 3 20.00` <br/> 
`0-201-70353-X 2 25.00` <br/>
则输出： <br/>
`0-201-70353-X 5 110 22`


3.  **改进Sales_data加法用例**



```cpp
#include <iostream>
#include "Sales_data.h"
int main()
{
    Sales_data item1, item2; 
    std::cin >> item1 >> item2; 
    // 首先检查 item1 和 item2 是否表示相同的书
    if (item1.isbn() == item2.isbn())
    {
        std::cout << item1 + item2 << std::endl; 
        return 0; 
    }
    else 
    {
        std::cerr << "Data must refer to same ISBN" << std::endl; 
        return -1;      // 表示失败
    }
}
```


4.  **书店程序用例**<br/>
现在假定我们已经准备好完成书店程序了。 我们需要从一个文件中读取销售记录，生成每本书的销售报告，显示销售册数，总销售额和平均售价。为了简单起见我们假定每个 ISBN 书号的所有销售记录在文件中是聚在一起保存的。 <br/>
**程序说明：**我们的程序会将每个 ISBN 的所有数据合并起来，存入名为 total 的变量中。 我们使用另一个名为 trans 的变量保存读取的每条销售记录。如果 trans 和 total 指向相同的 ISBN, 我们会更新 total 的值。 否则，我们会打印 total 的值， 并将其重置为刚刚读取的数据(trans):

```cpp
#include <iostream>
#include "Sales_data.h"
int main()
{
    Sales_data total;   // 保存下一条交易记录的变量
    // 读入第一条交易记录，并确保有数据可以处理
    if (read(cin, total))
    {
        Sales_data trans;   // 保存下一条记录的变量
        // 读入并处理剩余交易记录
        while (read(trans))
        {
            // 如果我们仍在处理相同的书
            if (total.isbn() == trans.isbn())
                total.combine(trans);     // 更新总销售额
            else
            {
                // 打印前一本书的结果
                print(cout, total) << std::endl; 
                total = trans;      // total 现在表示下一本书的销售额
            }
        }
        print(cout, total) << endl;     // 输出最后一条交易
    }
    else
    {
        // 没有输入！ 警告读者
        std::cerr << "No data?" << std::endl; 
    }
    return 0; 
}
```

### 接口
综上所述，Sales_data 的接口应该包含以下操作： 
- 一个 **isbn** 成员，用于返回对象的 ISBN 编号。 
- 一个 **combine 成员函数**，用于将一个 Sales_data 对象加到另一个对象上。 
- 一个名为 **add 的函数**，执行两个 Sales_data 对象的加法。 
- 一个 **read 函数**，将数据从 istream 读入到 Sales_data 对象中。 
- 一个 **print 函数**， 将 Sales_data 对象的值输出到 ostream。 

### 成员及其访问权限
- `private string bookNo`
- `private unsigned units_sold = 0`
- `private double revenue = 0.0` 

### 所有函数
- 成员接口函数
    + `string isbn() const`
    + `Sales_data& combine(const Sales_data&)`
    + `double avg_price() const`
- 非成员接口函数
    - `add()`
    - `read()`
    - `print()`

- ## 定义 Sales_data 的构造函数
    - 一个 istream&, 从中读取一条交易信息
    - 一个 const string&, 表示 IBSN编号； 一个 unsigned, 表示售出的图书数量； 以及一个 double，表示图书的售出价格。 
    - 一个 const string&, 表示 ISBN编号；编译器将赋予其他成员默认值。 
    - 一个空参数列表（即默认构造函数）
### 实现类
- 类定义

```cpp
#ifndef SALES_DATA_H
#define SALES_DATA_H
#include <iostream>
#include <string>

class Sales_data
{
public:
// 新增的构造函数
//Sales_data() = default; // 默认构造函数
//    Sales_data(std::string &s = "") : bookNo(s) { }
//    Sales_data(const std::string &s, unsigned n, double p):
//            bookNo(s), units_sold(n), revenue(p*n) { }
//    Sales_data(std::istream&);

// 委托构造函数
// 非委托构造函数使用对应的实参初始化成员
Sales_data(std::string s, unsigned cnt, double price):
    bookNo(s), units_sold(cnt), revenue(price * cnt) { }
// 其余构造函数全都委托给另一个构造函数
Sales_data() : Sales_data("", 0, 0){ }
Sales_data(std::string s) : Sales_data(s, 0, 0) { }
Sales_data(std::istream &is);// 委托给第二个构造函数

    // 新成员：关于 Sales_data 对象的操作
    std::string isbn() const { return bookNo; }
    Sales_data& combine(const Sales_data&);
    double avg_price() const;
    // 数据成员
    std::string bookNo;
    unsigned int units_sold = 0;
    double revenue = 0.0;
};

// Sales_data 的非成员接口函数
Sales_data add(const Sales_data&, const Sales_data&);
std::ostream &print(std::ostream&, const Sales_data&);
std::istream &read(std::istream&, Sales_data&);


#endif // SALES_DATA_H 
```

---

- 类实现

```cpp
#include "Sales_data.h"
// 在类的外部定义成员函数
Sales_data::Sales_data(std::istream &is)
{
    read(is, *this);
}

double Sales_data::avg_price() const
{
    if (units_sold)
        return revenue / units_sold;
    else
        return 0;
}

Sales_data& Sales_data::combine(const Sales_data &rhs)
{
    units_sold += rhs.units_sold;
    revenue += rhs.revenue;
    return *this;
}

// 在类的外部定义 类 Sales_data 的非成员接口函数
Sales_data add(const Sales_data &lhs, const Sales_data &rhs)
{
    Sales_data sum = lhs;       // 把 lhs 的数据成员拷贝给 sum
    sum.combine(rhs);           // 把 rhs 的数据成员加到 sum 当中
    return sum;
}
std::ostream &print(std::ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold << " "
        << item.revenue << " " << item.avg_price();
    return os;
}

std::istream &read(std::istream &is, Sales_data &item)
{
    double price = 0;
    is >> item.bookNo >> item.units_sold >> price;
    item.revenue = price * item.units_sold;
    return is;
}


```



