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
文件注释内容：

    版权说明、描述信息、生成日期、修改历史。

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

----
类和接口的注释：

    该注释放在 package 关键字之后，class 或者 trait 关键字之前。

> 说明：方便JavaDoc收集。
> 
> 示例：

```scala
package com.avcdata.streaming.evaluationbutler.processor

/**
 * 注释内容 
 */
class EvaluationButlerProcessor
```

----
类和接口的注释内容：

    类的注释主要是一句话功能简述、功能详细描述。

> 说明：可根据需要列出：版本号、生成日期、作者、内容、功能、与其它类的关系等。如果一个类存在Bug，请如实说明这些Bug。
> 
> 格式：

```
/**
 * 〈一句话功能简述〉  
 * 〈功能详细描述〉  
 * @author      [作者]   
 * @version     [版本号, YYYY-MM-DD]  
 * @see         [相关类/方法]  
 * @since       [产品/模块版本]   
 * @deprecated  
 */
```  

> 说明：描述部分说明该类或者接口的功能、作用、使用方法和注意事项，每次修改后增加作者和更新版本 号和日期，@since 表示从那个版本开始就有这个类或者接口，@deprecated 表示不建议使用该类或者接口。

> 示例：

```
/**
 * LogManager 类集中控制对日志读写的操作。   
 * 全部为静态变量和静态方法，对外提供统一接口。分配对应日志类型的读写器，  
 * 读取或写入符合条件的日志纪录。  
 * 
 * @author      张三，李四，王五 
 * @version     1.2, 2015-03-25  
 * @see         LogIteraotor  
 * @see         BasicLog  
 * @since       CommonLog1.0   
 */
```

----
类属性、公有和保护方法注释：
    
    写在类属性、公有和保护方法上面。

> 示例：

```
/**
 * 注释内容
 */
private val logType:String
 
/**
 * 注释内容  
 */  
def write(): Unit
```

----
成员变量注释内容：

    成员变量的意义、目的、功能，可能被用到的地方。
    
----
公有和保护方法注释内容：

    列出方法的一句话功能简述、功能详细描述、输入参数、输出参数、返回值、违例 等。  
    
> 格式：

```
/**
 * 〈一句话功能简述〉  
 * 〈功能详细描述〉   
 * @param [参数1]     [参数1说明]  
 * @param [参数2]     [参数2说明]  
 * @return           [返回类型说明]   
 * @exception/throws [违例类型] [违例说明]  
 * @see              [类、类#方法、类#成员]  
 * @deprecated  
 */
```

> 说明：@since 表示从那个版本开始就有这个方法；@exception或throws 列出可能仍出的异常；@deprecated  表示不建议使用该方法。

> 示例：

```
/**
 * 根据日志类型和时间读取日志。
 * 分配对应日志类型的LogReader，指定类型、查询时间段、条件和反复器缓冲数，
 * 读取日志记录。查询条件为null或0的表示没有限制，反复器缓冲数为0读不到日志。
 * 查询时间为左包含原则，即 [startTime, endTime) 。
 * @param logTypeName   日志类型名（在配置文件中定义的）
 * @param startTime     查询日志的开始时间      
 * @param endTime       查询日志的结束时间      
 * @param logLevel      查询日志的级别      
 * @param userName      查询该用户的日志      
 * @param bufferNum     日志反复器缓冲记录数
 * @return              结果集，日志反复器      
 * @since CommonLog1.0       
 */ 
public static LogIterator read(String logType, Date startTime,  Date endTime,
int logLevel, String userName, int bufferNum)
```

----
对于方法内部用throw语句抛出的异常，必须在方法的注释中标明，对于所调用的其他方法所抛出的异常，选择主要的在注释中说明。**对于非RuntimeException，即throws子句声明会抛出的异常，必须在方法的注释中标明。** 

> 说明：异常注释用@exception或@throws表示，在JavaDoc中两者等价，但推荐用@exception标注Runtime异常，@throws标注非Runtime异常。异常的注释必须说明该异常的含义及什么条件下抛出该异常。

----
注释应与其描述的代码相近，对代码的注释应放在其上方或右方（对单条语句的注释）相邻位置，不可放在下面，如放于上方则需与其上面的代码用空行隔开。

----
注释与所描述内容进行同样的缩排。

> 说明：可使程序排版整齐，并方便注释的阅读与理解。
> 
> 示例：如下例子，排版不整齐，阅读稍感不方便。

```scala
def example(): Unit
{
// 注释
    val One: CodeBlock
    
        // 注释     
    val Two: CodeBlock
}
```

> 应改为如下布局。

```scala
def example(): Unit
{
    // 注释
    val One: CodeBlock
    
    // 注释     
    val Two: CodeBlock
}
```

----
将注释与其上面的代码用空行隔开。

> 示例：如下例子，显得代码过于紧凑。

```
//注释  
program code one 
//注释  
program code two
```

> 应如下书写：

```
//注释  
program code one

//注释  
program code two
```

----
对变量的定义和分支语句（条件分支、循环语句等）必须编写注释。

> 说明：这些语句往往是程序实现某一特定功能的关键，对于维护人员来说，良好的注释帮助更好的理解程 序，有时甚至优于看设计文档。

----
边写代码边注释，修改代码同时修改相应的注释，以保证注释与代码的一致性。不再有用的注释要删除。

----
注释的内容要清楚、明了，含义准确，防止注释二义性。

> 说明：错误的注释不但无益反而有害。

----
避免在注释中使用缩写，特别是不常用缩写。

> 说明：在使用缩写时或之前，应对缩写进行必要的说明。

### 建议

避免在一行代码或表达式的中间插入注释。

> 说明：除非必要，不应在代码或表达中间插入注释，否则容易使代码可理解性变差。

----
通过对函数或过程、变量、结构等正确的命名以及合理地组织代码的结构，使代码成为自注释的。

> 说明：清晰准确的函数、变量等的命名，可增加代码可读性，并减少不必要的注释。 

----
在代码的功能、意图层次上进行注释，提供有用、额外的信息。

> 说明：注释的目的是解释代码的目的、功能和采用的方法，提供代码以外的信息，帮助读者理解代码，防 止没必要的重复注释信息。
> 
> 示例：如下注释意义不大。

```
// 如果 receiveFlag 为真 
if (receiveFlag)   
```

> 而如下的注释则给出了额外有用的信息。

```
// 如果从连结收到消息  
if (receiveFlag)
```

----
在程序块的结束行右方加注释标记，以表明某程序块的结束。

> 说明：当代码段较长，特别是多重嵌套时，这样做可以使代码更清晰，更便于阅读。

> 示例：

```
if (...)
{      
    program code1
    while (index < MAX_INDEX)
    {
        program code2
    } // end of while (index < MAX_INDEX)  // 指明该条while语句结束
} // end of if (...) // 指明是哪条if语句结束
```

----
注释应考虑程序易读及外观排版的因素，使用的语言若是中、英兼有的，建议多使用中文，除非能用非常流利准确的英文表达。

> 说明：注释语言不统一，影响程序易读性和外观排版，出于对维护人员的考虑，建议使用中文。

----
方法内的单行注释使用 //。  

> 说明：调试程序的时候可以方便的使用 /*...*/ 注释掉一长段程序。

----
注释尽量使用中文注释和中文标点。方法和类描述的第一句话尽量使用简洁明了的话概括一下功能，然后加以句号。接下来的部分可以详细描述。

> 说明：JavaDoc工具收集简介的时候使用选取第一句话。

----
顺序实现流程的说明使用1、2、3、4在每个实现步骤部分的代码前面进行注释。

> 示例：如下是对设置属性的流程注释。

```
//1、 判断输入参数是否有效。
。。。。。

// 2、设置本地变量。
。。。。。
```

----
一些复杂的代码需要说明。

> 示例：这里主要是对闰年算法的说明。

```
//1. 如果能被4整除，是闰年；
//2. 如果能被100整除，不是闰年；
//3. 如果能被400整除，是闰年。
```

## 命名规范

