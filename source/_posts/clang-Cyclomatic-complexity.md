---
title: Clang 插件计算圈复杂度
date: 2020-04-01 00:02:12
tags: [编译原理]
---

![](http://qiniu.huyangjie.cn/mweb/15853651754787.jpg)

<!-- more -->

## 圈复杂度
>圈复杂度（Cyclomatic complexity，简写CC）也称为条件复杂度，是一种代码复杂度的衡量标准。由托马斯·J·麦凯布（Thomas J. McCabe, Sr.）于1976年提出，用来表示程序的复杂度，其符号为VG或是M。它可以用来衡量一个模块判定结构的复杂程度，数量上表现为独立现行路径条数，也可理解为覆盖所有的可能情况最少使用的测试用例数。圈复杂度大说明程序代码的判断逻辑复杂，可能质量低且难于测试和 维护。程序的可能错误和高的圈复杂度有着很大关系。
>
>通俗得讲圈复杂度就是确定某一方法到达100%的覆盖率将需要多少条测试用例
>
  References:
  - McCabe (December 1976). “A Complexity Measure”.
    IEEE Transactions on Software Engineering: 308–320
    http://www.literateprogramming.com/mccabe.pdf
 > 


## 圈复杂度算法

### 点边计算法

计算公式

    V(G) = e – n + 2 * p
    
>e：控制流图中边的数量（对应代码中顺序结构的部分）
n：代表在控制流图中的判定节点数量，包括起点和终点（对应代码中的分支语句）
ps：所有终点只计算一次，即使有多个 return 或者 throw


>p：独立组件的个数


### 节点计算法

圈复杂度实际上就是等于判定节点的数量再加上1，也即控制流图的区域数，对应的计算公式为：

    V (G) = P + 1
    
其中P为判定节点数，判定节点举例：

* if语句
* while语句
* for语句
* case语句
* catch语句
* and和or布尔操作
* ?:三元运算符

常见语句的控制流：
![](http://qiniu.huyangjie.cn/mweb/15845138142623.jpg)



1、所有终点只计算一次，即便有多个return或者throw；
2、节点对应代码中的分支语句

使用Clang插件代码实现自动检测的话，我们使用节点计算法

### <span id="func">小结</span>

如[高复杂度的代码是否可维护性差](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/#%E6%80%9D%E8%BE%A81-%E9%AB%98%E5%A4%8D%E6%9D%82%E5%BA%A6%E7%9A%84%E4%BB%A3%E7%A0%81%E6%98%AF%E5%90%A6%E5%8F%AF%E7%BB%B4%E6%8A%A4%E6%80%A7%E5%B7%AE) 圈复杂度高的情况也并不是需要重构

如[复杂度相同的代码是否是一致的](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/#%E6%80%9D%E8%BE%A82-%E5%A4%8D%E6%9D%82%E5%BA%A6%E7%9B%B8%E5%90%8C%E7%9A%84%E4%BB%A3%E7%A0%81%E6%98%AF%E5%90%A6%E6%98%AF%E4%B8%80%E8%87%B4%E7%9A%84) 圈复杂度相同的情况代码健康状况也不同

圈复杂度还需要具体情况具体分析，其只能作为重构的一个度量指标，作为决策的一个参考依据。

降低圈复杂度不在本文讨论范围，可以点击 [降低圈复杂度方法](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/#%E9%99%8D%E4%BD%8E%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6%E7%9A%84%E6%96%B9%E6%B3%95) 链接查看

## Clang 插件逻辑

### Clang命令集
1. clang分析AST

    使用命令：`clang -Xclang -ast-dump -fsyntax-only xxx.m` ，即可分析xxx.m 的 AST
    
    ```
    clang -Xclang -ast-dump -fsyntax-only jake.m
    ```

2. clang-query作为clang的一个工具，可交互式检验Matcher正确性和有效性，可探索AST的结构和关系。

    ```
    ~/clang-llvm/build/bin/clang-query jake.m
    ```
    
### Clang 圈复杂度算法实现

文中的两种圈复杂度计算方法，点边计算对人更加直观，不过在代码中实现，用节点计算法对计算机而言才是更直观的方法。

```
 V (G) = P + 1
```

只需要统计一个模块（函数）中所有P出现的次数，具体在Clang中可以重载`RecursiveASTVisitor` 类（此类用于搜索整个AST，并访问每一个节点），然后重载里面的这些方法

``` c++
bool VisitIfStmt(clang::IfStmt *stmt);
bool VisitForStmt(clang::ForStmt *stmt);
bool VisitObjCForCollectionStmt(clang::ObjCForCollectionStmt *stmt);
bool VisitWhileStmt(clang::WhileStmt *stmt);
bool VisitDoStmt(clang::DoStmt *stmt);
bool VisitCaseStmt(clang::CaseStmt *stmt);
bool VisitObjCAtCatchStmt(clang::ObjCAtCatchStmt *stmt);
bool VisitCXXCatchStmt(clang::CXXCatchStmt *stmt);
bool VisitConditionalOperator(clang::ConditionalOperator *op);
bool VisitBinaryOperator(clang::BinaryOperator *op);
```


在重载的方法内对此模块中节点P的次数进行一次累加并返回true，如:

``` c
bool VisitIfStmt(clang::IfStmt *stmt) {
    _count++;
    return true;
}
```

遍历一次模块语法树之后获得此模块的圈复杂度值，再在另一个`RecursiveASTVisitor`类中进行判断，一般圈复杂度不要超过10，以下规范来自网络仅做参考

|  圈复杂度 | 代码状况  |  可测性 | 维护成本|
|---|---|---|---|
|1-10	|清晰、结构化|	高|	低|
|10-20|	复杂|	中	|中|
|20-30|	非常复杂|	低|	高|
|30|	不可读	|不可测|非常高|

在模块编译完成后，将会在模块的结尾进行提示，类似形式如下：
![](http://qiniu.huyangjie.cn/mweb/15851935710272.jpg)



### 思考

如下是一个圈复杂度大于10的AST语法树示例：

```
CompoundStmt 0x7ff7f2898788
|-CallExpr 0x7ff7f289af60 'void'
| |-ImplicitCastExpr 0x7ff7f289af48 'void (*)(id, ...)' <FunctionToPointerDecay>
| | `-DeclRefExpr 0x7ff7f289ae60 'void (id, ...)' Function 0x7ff7f2055540 'NSLog' 'void (id, ...)'
| `-ImplicitCastExpr 0x7ff7f289af88 'id':'id' <BitCast>
|   `-ObjCStringLiteral 0x7ff7f289aed0 'NSString *'
|     `-StringLiteral 0x7ff7f289aeb8 'char [1]' lvalue ""
|-DeclStmt 0x7ff7f289b040
| `-VarDecl 0x7ff7f289afb8  used num 'int' cinit
|   `-IntegerLiteral 0x7ff7f289b020 'int' 0
|-IfStmt 0x7ff7f289b238 has_else
| |-BinaryOperator 0x7ff7f289b0c8 'int' '=='
| | |-ImplicitCastExpr 0x7ff7f289b0b0 'int' <LValueToRValue>
| | | `-DeclRefExpr 0x7ff7f289b058 'int' lvalue Var 0x7ff7f289afb8 'num' 'int'
| | `-IntegerLiteral 0x7ff7f289b090 'int' 0
| |-CompoundStmt 0x7ff7f289b1d8
| | `-IfStmt 0x7ff7f289b1c0
| |   |-BinaryOperator 0x7ff7f289b158 'int' '=='
| |   | |-ImplicitCastExpr 0x7ff7f289b140 'int' <LValueToRValue>
| |   | | `-DeclRefExpr 0x7ff7f289b0e8 'int' lvalue Var 0x7ff7f289afb8 'num' 'int'
| |   | `-UnaryOperator 0x7ff7f289b128 'int' prefix '-'
| |   |   `-IntegerLiteral 0x7ff7f289b108 'int' 1
| |   `-CompoundStmt 0x7ff7f289b1a8
| |     `-ReturnStmt 0x7ff7f289b198
| |       `-IntegerLiteral 0x7ff7f289b178 'int' 0
| `-CompoundStmt 0x7ff7f289b220
|   `-ReturnStmt 0x7ff7f289b210
|     `-IntegerLiteral 0x7ff7f289b1f0 'int' 0
|-DeclStmt 0x7ff7f289b300
| `-VarDecl 0x7ff7f289b278  used index 'int' cinit
|   `-IntegerLiteral 0x7ff7f289b2e0 'int' 0
|-SwitchStmt 0x7ff7f289b368
| |-ImplicitCastExpr 0x7ff7f289b350 'int' <LValueToRValue>
| | `-DeclRefExpr 0x7ff7f289b318 'int' lvalue Var 0x7ff7f289b278 'index' 'int'
| `-CompoundStmt 0x7ff7f2898700
|   |-CaseStmt 0x7ff7f289b3c0
|   | |-ConstantExpr 0x7ff7f289b3a8 'int'
|   | | `-IntegerLiteral 0x7ff7f289b388 'int' 1
|   | `-BreakStmt 0x7ff7f289b3e8
|   |-CaseStmt 0x7ff7f2898438
|   | |-ConstantExpr 0x7ff7f2898420 'int'
|   | | `-IntegerLiteral 0x7ff7f2898400 'int' 2
|   | `-BreakStmt 0x7ff7f2898460
|   |-CaseStmt 0x7ff7f28984a0
|   | |-ConstantExpr 0x7ff7f2898488 'int'
|   | | `-IntegerLiteral 0x7ff7f2898468 'int' 3
|   | `-BreakStmt 0x7ff7f28984c8
|   |-CaseStmt 0x7ff7f2898508
|   | |-ConstantExpr 0x7ff7f28984f0 'int'
|   | | `-IntegerLiteral 0x7ff7f28984d0 'int' 4
|   | `-BreakStmt 0x7ff7f2898530
|   |-CaseStmt 0x7ff7f2898570
|   | |-ConstantExpr 0x7ff7f2898558 'int'
|   | | `-IntegerLiteral 0x7ff7f2898538 'int' 5
|   | `-BreakStmt 0x7ff7f2898598
|   |-CaseStmt 0x7ff7f28985d8
|   | |-ConstantExpr 0x7ff7f28985c0 'int'
|   | | `-IntegerLiteral 0x7ff7f28985a0 'int' 6
|   | `-BreakStmt 0x7ff7f2898600
|   |-CaseStmt 0x7ff7f2898640
|   | |-ConstantExpr 0x7ff7f2898628 'int'
|   | | `-IntegerLiteral 0x7ff7f2898608 'int' 7
|   | `-BreakStmt 0x7ff7f2898668
|   |-CaseStmt 0x7ff7f28986a8
|   | |-ConstantExpr 0x7ff7f2898690 'int'
|   | | `-IntegerLiteral 0x7ff7f2898670 'int' 8
|   | `-BreakStmt 0x7ff7f28986d0
|   `-DefaultStmt 0x7ff7f28986e0
|     `-BreakStmt 0x7ff7f28986d8
`-ReturnStmt 0x7ff7f2898778
  `-IntegerLiteral 0x7ff7f2898758 'int' 0
```

其源码为：

``` objectivec
- (int)func {
    NSLog(@"");
    int num = 0;
    if (num == 0) {
        if (num == -1) {
            return 0;
        }
    } else {
        return 0;
    }
    int index = 0;
    switch (index) {
        case 1:
            
            break;
        case 2:
            
            break;
        case 3:
            
            break;
        case 4:
            
            break;
        case 5:
            
            break;
        case 6:
            
            break;
        case 7:
            
            break;
        case 8:
            
            break;
            
        default:
            break;
    }
    return 0;
}
```

[高复杂度的代码是否可维护性差](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/#%E6%80%9D%E8%BE%A81-%E9%AB%98%E5%A4%8D%E6%9D%82%E5%BA%A6%E7%9A%84%E4%BB%A3%E7%A0%81%E6%98%AF%E5%90%A6%E5%8F%AF%E7%BB%B4%E6%8A%A4%E6%80%A7%E5%B7%AE)

## Reference

[ Clang 官方文档](https://clang.llvm.org/doxygen/classclang_1_1PluginASTAction.html#a738fc8000ed0a254d23fb44f0fd1d54c)

[详解圈复杂度](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/
)

[美团-代码质量管控 -- 复杂度检测](https://juejin.im/post/59bb8b546fb9a00a4247532e#heading-2)

[OCLint Docs](https://oclint-docs.readthedocs.io/en/stable/rules/size.html)

https://blog.csdn.net/vincentiss/article/details/54617915

[基于clang插件的一种iOS包大小瘦身方案](https://mp.weixin.qq.com/s?__biz=MzUxMzcxMzE5Ng==&mid=2247488360&amp;idx=1&amp;sn=94fba30a87d0f9bc0b9ff94d3fed3386&source=41#wechat_redirect)

[LLVM 官方 ExternalClangExamples](http://clang.llvm.org/docs/ExternalClangExamples.html)

[ASTMatcher分析函数调用链](https://cloud.tencent.com/developer/article/1523137)