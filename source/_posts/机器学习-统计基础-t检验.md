---
title: 机器学习-统计基础-t检验
date: 2018-02-08 11:09:26
tags: [Machine Learning]
categories: [学习笔记]
---

### 概念记录

#### t分布是用自由度来定义

##### 自由度（*Degrees of freedom*）
 
 自由度-选择n个数字: 
 `自由度是n`
 
 自由度-添加至10: 
  `有n个数字相加等于10，那自由度就为n-1`

 自由度-边际总数: 
  `（n*n）的表格自由度就有(n-1)*(n-1)`
  
  
  随着自由度的增大，t分布会越来越接近于正态分布
  
  <!--more-->
  
#### t表格
t表显示的是临界值，左边显示的是自由度，右边（顶部）显示的是右尾的面积， 底部的是置信区间
t表格

<img src="http://qiniu.huyangjie.cn/article/img/E9D9E242DE8D75E679D0A8140E93FE85.jpg" width="300px">
[大图点击这里](https://s3.amazonaws.com/udacity-hosted-downloads/t-table.jpg)


#### 影响t统计量

<img src="http://qiniu.huyangjie.cn/article/img/DC58EA0C446C0CE76C23869C22D51F60.jpg" width="300px">

将t分布的中心放在 μ0 处，然后看看x拔位于这个分布的那个位置，x拔越靠近两端，更有可能所来自的总体的均值和  μ0 显著不同。

#### P值

P值等于t统计量在红色的区域内的概率

<img src="http://qiniu.huyangjie.cn/article/img/908A45D67C9EA2F4066FC4ECCEC4D899.jpg" width="300px">

当P值小于α水平时，我们会拒绝零假设

使用[GraphPad](http://www.graphpad.com/quickcalcs/)来找到P值

*【疑问】*

[统计显著性](#statistically_significant) (*statistically significant*)
根据一般规则，通常α水平等于0.05， 这一差别不具有统计显著性

拒绝零假设，代表具有统计显著性

#### Cohen's d
以统计学家Jacob Cohen命名

Cohen's d 是一种[效应量](#effect_size)度量用来衡量两个均值之间的标准化均值差（以标准偏差为单位）， 除以的是样本标准差

<img src="http://qiniu.huyangjie.cn/article/img/1E97F29DD3AF56E94DB5ABE140BF602C.jpg" width="300px">


#### 置信区间
`(x̅-t*S/√n， x̅+t*S/√n）`

#### 误差范围
`t*S/√n`

#### 相依样本
如果同意受试者参加两次测试，则这两次结果是相依样本。为了衡量这些值之间的差别 Di = xi - yi

μQ - μA = μD
<img src="http://qiniu.huyangjie.cn/article/img/716D7E3BF12230B87FAF1AEE27E1646C.jpg" width="300px">

<img src="http://qiniu.huyangjie.cn/article/img/E14D2A3F524834B0A3A21CF36082E572.jpg" width="300px">

#### 相依样本t检验 设计类型
10a 31

* __重复衡量设计(Two conditions)__

  检验并衡量不同条件下的样本

    零假设是指这两个条件下的均值将相同

* __纵向设计（Growth over time *[longitudinal study]* ）__

    在一个时间段衡量某个变量，在晚些时候的某个时间点再衡量同一变量
    
    零假设也是两个总体均值相同，跟重复衡量设计不同的是在衡量变量时中间间隔了很长时间
    
* __预期检验和后期检验(pre-test, post-test)__

    先衡量某个变量，然后进行某种处理，然后在处理之后再对同一样本衡量同一变量，看看处理措施是否导致了显著的效应
    
    零假设是保持不变，即在处理前和处理后改变量没有出现显著的变化
    
   
#### 相依样本的优缺点

__Advantages__
* 控制个体差异性
  * 使用更少的受试者
  * 成本更低
  * 花费时间更少
  * 开支更少
  
__Disadvantages__
* 残留效应（因为控制个体差异性，使个体参与了两次试验，第一次试验可能会影响到第二次的结果）
* 试验的处理顺序可能会影响到结果
    
#### 独立样本
独立样本的优势就是相依样本的劣势，劣势是相依样本的优势

开展实验性检验对受试者实施处理措施，或开展观察性检验，我们只是观察两组不同总体的特性

####  <span id = effect_size>效应量</span>
在实验性研究中，或存在处理变量的研究中，效应量是指处理效应的大小
在非实验性研究中，效应量是指变量之间的关系强度，在z检验或者t检验中，最简单的效应量衡量指标是均值差异

#### <span id = statistically_significant>统计显著性</span>（*statistically significant*）
统计显著性知识表示结果可能不是偶然发生的，在解释结果时排除了随机因素或抽样错误

##### 如何判断某个调查研究的结果是否有意义
1. 度量的是什么？
2. 效应有多大？
3. 在解释结果时能排除随机因素吗？
4. 解释结果时能排除潜在变量吗？

#### r^2
r^2表示的是两个变量之间的关系程度也称为确定系数。
r^2是一个比例，范围从0到1. 0代表两个变量根本没有关系，1代表两个变量完全相关

<img src="http://qiniu.huyangjie.cn/article/img/CBE24A42655F62B672824B2419B2E453.jpg" width="300px">
这里的t不是临界值，是从t检验中获得的值

#### 报告结果

APA Style

t检验
`t(df) = X.XX, p=.XX, direction`
t值，p值，单尾检验还是双尾检验

置信区间
`95% CI = （4,6）`

效应量
`d = X.XX, r^2 = .XX`

#### t检验公式

<img src="http://qiniu.huyangjie.cn/article/img/4F58FD2662042864DE48F2B39220549D.jpg" width="300px">
<img src="http://qiniu.huyangjie.cn/article/img/CFA3D97A5FD10F5092E3D69683577632.jpg" width="300px">


