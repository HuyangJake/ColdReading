---
title: 降低圈复杂度的方法
date: 2020-04-01 00:06:30
tags: [编译原理]
---

## 重新组织你的函数
### 技巧1 提炼函数
有一段代码可以被组织在一起并独立出来:

``` c
void Example(int val)
{
	if( val > MAX_VAL)
	{
		val = MAX_VAL;
	}

	for( int i = 0; i < val; i++)
	{
		doSomething(i);
	}
}
```
<!-- more -->

将这段代码放进一个独立函数中，并让函数名称解释该函数的用途:

``` c
int getValidVal(int val)
{
   	if( val > MAX_VAL)
	{
		return MAX_VAL;
	} 
    return val;
}

void doSomethings(int val)
{
	for( int i = 0; i < val; i++)
	{
		doSomething(i);
	}
}

void Example(int val)
{
    doSomethings(getValidVal(val));
}
``` 
最后还要重新审视函数内容是否在统一层次上。

### 技巧2 替换算法
把某个算法替换为另一个更清晰的算法：

``` c
string foundPerson(const vector<string>& peoples){
  for (auto& people : peoples) 
  {
    if (people == "Don"){
      return "Don";
    }
    if (people == "John"){
      return "John";
    }
    if (people == "Kent"){
      return "Kent";
    }
  }
  return "";
}
```

将函数实现替换为另一个算法:

``` c
string foundPerson(const vector<string>& people){
  std::map<string,string>candidates{
    	{ "Don", "Don"},
    	{ "John", "John"},
    	{ "Kent", "Kent"},
       };
  for (auto& people : peoples) 
  {
    auto& it = candidates.find(people);
    if(it != candidates.end())
        return it->second;
  }
}
```

所谓的表驱动。

## 简化条件表达式
### 技巧3 逆向表达
在代码中可能存在条件表达如下：

``` c
if ((condition1() && condition2()) || !condition1())
{
    return true;
}
else
{
    return false;
}
应用逆向表达调换表达顺序后效果如下：

if(condition1() && !condition2())
{
    return false;
}

return true;
```

### 技巧4 分解条件
在代码中存在复杂的条件表达：

``` c
if(date.before (SUMMER_START) || date.after(SUMMER_END))
    charge = quantity * _winterRate + _winterServiceCharge;
else 
    charge = quantity * _summerRate;
从if、then、else三个段落中分别提炼出独立函数：

if(notSummer(date))
    charge = winterCharge(quantity);
else 
    charge = summerCharge (quantity);
    
```
    
### 技巧5 合并条件
一系列条件判断，都得到相同结果:

``` c
double disabilityAmount() 
{
    if (_seniority < 2) return 0;
    if (_monthsDisabled > 12) return 0;
    if (_isPartTime) return 0;
    // compute the disability amount     ......
将这些判断合并为一个条件式，并将这个条件式提炼成为一个独立函数:

double disabilityAmount() 
{
    if (isNotEligableForDisability()) return 0;
    // compute the disability amount
    ......
```    

### 技巧6 移除控制标记
在代码逻辑中，有时候会使用bool类型作为逻辑控制标记：

``` c
void checkSecurity(vector<string>& peoples) {
	bool found = false;
	for (auto& people : peoples) 
    {
		if (! found) {
			if (people == "Don"){
				sendAlert();
				found = true;
			}
			if (people == "John"){
				   sendAlert();
				   found = true;
			}
		}
	}
}
```

使用break和return取代控制标记：

``` c
void checkSecurity(vector<string>& peoples) {
	for (auto& people : peoples)
	{     
		if (people == "Don" || people == "John")
		{
			sendAlert();
			break;
		}
	}
}
```

### 技巧7 以多态取代条件式
条件式根据对象类型的不同而选择不同的行为：

``` c
double getSpeed() 
{
    switch (_type) {
        case EUROPEAN:
            return getBaseSpeed();
        case AFRICAN:
            return getBaseSpeed() - getLoadFactor() *_numberOfCoconuts;
        case NORWEGIAN_BLUE:
            return (_isNailed) ? 0 : getBaseSpeed(_voltage);
    }
    throw new RuntimeException ("Should be unreachable");
}
```

将整个条件式的每个分支放进一个子类的重载方法中，然后将原始函数声明为抽象方法：

``` c
class Bird
{
public:
    virtual double getSpeed() = 0;
    
protected:
    double getBaseSpeed();
}

class EuropeanBird
{
public:
    double getSpeed()
    {
        return getBaseSpeed();
    }
}

class AfricanBird
{
public:
    double getSpeed()
    {
        return getBaseSpeed() - getLoadFactor() *_numberOfCoconuts;
    }
    
private:
    double getLoadFactor();
    
    double _numberOfCoconuts;
}

class NorwegianBlueBird
{
public:
    double getSpeed()
    {
        return (_isNailed) ? 0 : getBaseSpeed(_voltage);
    };
    
private:
    bool _isNailed;
}
```

## 简化函数调用
### 技巧8 读写分离
某个函数既返回对象状态值，又修改对象状态:

``` c
class Customer
{
int getTotalOutstandingAndSetReadyForSummaries(int number);
}
建立两个不同的函数，其中一个负责查询，另一个负责修改:

class Customer
{
    int getTotalOutstanding();
    void SetReadyForSummaries(int number);
}
```

### 技巧9 参数化方法
若干函数做了类似的工作，但在函数本体中却 包含了不同的值:

``` c
Dollars baseCharge()
 {
    double result = Math.min(lastUsage(),100) * 0.03;
    if (lastUsage() > 100)
    {
        result += (Math.min (lastUsage(),200) - 100) * 0.05;
    }
    if (lastUsage() > 200)
    {
        result += (lastUsage() - 200) * 0.07;
    }
    return new Dollars (result);
}
```

建立单一函数，以参数表达那些不同的值：

``` c
Dollars baseCharge() 
{
    double result = usageInRange(0, 100) * 0.03;
    result += usageInRange (100,200) * 0.05;
    result += usageInRange (200, Integer.MAX_VALUE) * 0.07;
    return new Dollars (result);
}

int usageInRange(int start, int end) 
{
    if (lastUsage() > start) 
        return Math.min(lastUsage(),end) -start;
     
    return 0;
}
```

### 技巧10 以明确函数取代参数
函数实现完全取决于参数值而采取不同反应：

``` c
void setValue (string name, int value) 
{
    if (name == "height")
        _height = value;
    else if (name == "width")
        _width = value;
    Assert.shouldNeverReachHere();
}
针对该参数的每一个可能值，建立一个独立函数：

void setHeight(int arg) 
{
    _height = arg;
}
void setWidth (int arg) 
{
    _width = arg;
}
```


-----

Reference： [详解圈复杂度](http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/#%E9%99%8D%E4%BD%8E%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6%E7%9A%84%E6%96%B9%E6%B3%95)