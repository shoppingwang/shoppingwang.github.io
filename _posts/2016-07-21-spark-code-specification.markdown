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

- 程序块要采用缩进风格编写，缩进的空格数为4个。

> **说明**：对于由开发工具自动生成的代码可以有不一致。

- 分界符（如大括号‘{’和‘}’）应各独占一行并且位于同一列，同时与引用它们的语句左对齐。在函数体的开始、类和接口的定义、以及if、for、do、while、try、catch语句中的程序都要采用如上的缩进方式。

> **示例**
> 
> 如下例子不符合规范。

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

- 较长的语句、表达式或参数（>80字符）要分成多行书写，长表达式要在低优先级操作符处划分新行，操作符放在新行之首，划分出的新行要进行适当的缩进，使排版整齐，语句可读。

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