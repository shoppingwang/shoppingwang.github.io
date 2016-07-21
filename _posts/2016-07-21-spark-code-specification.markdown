---
layout:     post
title:      "Spark Job Code Specification"
subtitle:   "."
date:       2016-07-21
author:     "xp"
header-img: "img/tag-bg.jpg"
header-mask: 0.3
catalog:    true
tags:
    - AVCData
    - Tool
---
* This will become a table of contents (this text will be scraped).
{:toc}


## 范围
- 本规范规定了使用Scala语言编程时排版、注释、命名的规则和建议。
- 本规范适用于使用Scala语言编程的部门和产品。


## 术语和定义
- **规则**：编程时必须强制遵守的原则。
- **建议**：编程时必须加以考虑的原则。
- **格式**：对此规范格式的说明。
- **说明**：对此规范或者建议进行必要的解释。
- **示例**：对此规范或建议从正、反两个方面给出例子。


## 排版规则

### 规则

程序块要采用缩进风格编写，缩进的空格数为4个。

> 说明：对于由开发工具自动生成的代码可以有不一致。

----
分界符（如大括号‘{’和‘}’）应各独占一行并且位于同一列，同时与引用它们的语句左对齐。在函数体的开始、类和接口的定义、以及if、for、do、while、try、catch语句中的程序都要采用如上的缩进方式。

> 示例：如下例子不符合规范。

```scala
for (...) {
    ... // program code 
}

if (...)
    {      
        ... // program code
    }

def example_fun():Unit
     {
           ... // program code
     }
```

> 应如下书写。

```scala
for (...)
{
    ... // program code
}

if (...)
{
    ... // program code
}

def example_fun():Unit
{
    ... // program code
}
```

----
较长的语句、表达式或参数（>80字符）要分成多行书写，长表达式要在低优先级操作符处划分新行，操作符放在新行之首，划分出的新行要进行适当的缩进，使排版整齐，语句可读。

> 示例

```scala
if (filename != null 
    && new File(logPath + filename).length() < LogConfig.getFileSize()) 
{
    ... // program code
}

public static LogIterator read(String logType, Date startTime, Date endTime, 
int logLevel, String userName, int bufferNum)
```

----
不允许把多个短语句写在一行中，即一行只写一条语句。

> 示例：如下例子不符合规范。

```scala
    val now: LogFilename = null;       val that: LogFilename = null
```

> 应如下书写

```scala
    val now: LogFilename = null
    val that: LogFilename = null
```

----
相对独立的程序块之间、变量说明之后必须加空行。

> 示例：如下例子不符合规范。

```scala
if(log.getLevel() < LogConfig.getRecordLevel())
{
    return
}
val writer: LogWriter = null
```

> 应如下书写

```scala
if(log.getLevel() < LogConfig.getRecordLevel())
{
    return
}

val writer: LogWriter = null
val index:Int = 0
```

----
对齐只使用空格键，不使用TAB键。

>说明：以免用不同的编辑器阅读程序时，因TAB键所设置的空格数目不同而造成程序布局不整齐。IntelliJ IDEA、JBuilder、 UltraEdit等编辑环境，支持行首TAB替换成空格，应将该选项打开。

----
在两个以上的关键字、变量、常量进行对等操作时，它们之间的操作符之前、之后或者前后要加空格；进行非对等操作时，如果是关系密切的立即操作符（如.），后不应加空格。

> 说明：采用这种松散方式编写代码的目的是使代码更加清晰。

由于留空格所产生的清晰性是相对的，所以，在已经非常清晰的语句中没有必要再留空格，如果语句已足 够清晰则括号内侧(即左括号后面和右括号前面)不需要加空格，多重括号间不必加空格，因为在Java语言中括号已经是最清晰的标志了。

在长语句中，如果需要加的空格非常多，那么应该保持整体清晰，而在局部不加空格。给操作符留空格时 不要连续留两个以上空格。

> 示例：

- 逗号、分号只在后面加空格。

    mainMatchFields: Array[String], secondMatchFields: Array[String]

- 比较操作符, 赋值操作符"="、 "+="，算术操作符"+"、"%"，逻辑操作符"&&"、"&"，位域操作符 "<<"、"^"等双目操作符的前后加空格。

    if (current_time >= MAX_TIME_VALUE)
    
    a = b + c
    
    a *= 2
    
    a = b ^ 2
    
- "!"、"~"、"++"、"--"、"&"（地址运算符）等单目操作符前后不加空格。

    flag = !isEmpty // 非操作"!"与内容之间 
    
    i++             // "++","--"与内容之间
    
- "."前后不加空格。

    p.id = pid     // "."前后不加空格
    
- if、for、while等与后面的括号间应加空格，使if等关键字更为突出、明显。

    if (a >= b && c > d)
    
    
### 建议
类属性和类方法不要交叉放置，不同存取范围的属性或者方法也尽量不要交叉放置。

> 格式：

```
类定义
{
    类的公有属性定义
    类的保护属性定义
    类的私有属性定义
    类的公有方法定义
    类的保护方法定义
    类的私有方法定义
}
```


## 注释规范

### 规则

一般情况下，源程序有效注释量必须在30％以上。

> 说明：注释的原则是有助于对程序的阅读理解，在该加的地方都加了，注释不宜太多也不能太少，注释语 言必须准确、易懂、简洁。可以用注释统计工具来统计。

----
包的注释：
    
    包的注释写入一个名为 package.html 的HTML格式的说明文件放入当前路径。

> 说明：方便JavaDoc收集。

> 示例：com/avcdata/streaming/evaluationbutler/processor/package.html

----
包的注释内容：
    
    简述本包的作用、详细描述本包的内容、产品模块名称和版本、公司版权。

> 说明：在详细描述中应该说明这个包的作用以及在整个项目中的位置。
> 
> 格式：

```html
<html>
    <body>
        <p>一句话简述。
        <p>详细描述。
        <p>产品模块名称和版本
        <br>公司版权信息
    </body>
</html>
```
----
文件注释：

    文件注释写入文件头部，包名之前的位置。
    
> 说明：注意以 /* 开始避免被 JavaDoc 收集。
> 
> 示例： 

```
/*
 * 注释内容 
 */ 
 
package com.avcdata.streaming.evaluationbutler.processor
```

----
文件注释内容：版权说明、描述信息、生成日期、修改历史。

> 说明：文件名可选。
> 
> 格式：

```
/*
 * 文件名：[文件名]
 * 版权：〈版权〉
 * 描述：〈描述〉
 * 修改人：〈修改人〉
 * 修改时间：YYYY-MM-DD
 * 跟踪单号：〈跟踪单号〉
 * 修改单号：〈修改单号〉
 * 修改内容：〈修改内容〉
 */
```