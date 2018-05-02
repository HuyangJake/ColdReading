---
title: CheckBox in React Native
date: 2018-04-05 11:54:23
tags: [React Native]
categories: [学习笔记]
---

### 关于RN CheckBox组件

在写hello world过程中iOS同学发现CheckBox组件展示不了，但是文档上确确实实是存在这个组件的介绍的，只不过没有demo。

我查了源码的`CheckBox`实现和发布说明发现以下信息，分享下：

`CheckBox` 这个组件是在 RN `v0.49.0`版本引入，这是`CheckBox`的首现版本的说明：

https://github.com/facebook/react-native/releases/tag/v0.49.0

<!--more-->

可以看到当时只是发布在了Android的新功能列表中


以下是`0.49.0`版本中关于`CheckBox`的源码：
https://github.com/facebook/react-native/commit/84b11dd5185c017a316510de6ace608dc82321e8


commit 记录中也说明是Android的功能

```
Add Android React Native Checkbox
Reviewed By: achen1
Differential Revision: D5281736
fbshipit-source-id: 9a3c93eeace2d80be4ddbd4ffc3258c1d3637480
```
-----

后来估计很多iOS的同学去使用了`CheckBox`，毕竟文档中没有特别说明只有Android可以使用(iOS同学使用会报错)。于是在`v0.50.0`版本的更新说明中，特地对`CheckBox`添加了注释，并修改文件名，加了`Android`字样🙃

```
 ../Libraries/Components/CheckBox/CheckBox.js
```

修改为：

```
'../Libraries/Components/CheckBox/CheckBox.android.js',
```
并且添加了一个新的文件iOS版本的`CheckBox`！喜出望外！

![](http://o8ajh91ch.bkt.clouddn.com/CheckBoxList.png)


结果iOS的实现如下：
![](http://o8ajh91ch.bkt.clouddn.com/CheckBox_iOS.png)

🤔🤔🤔思考人生...🤔🤔🤔


以下是这次（`0.50.0`）关于`CheckBox`的commit信息

```

Use UnimplementedView for CheckBox on iOS

Summary:
`CheckBox` component was introduced in v0.49.0 and not implemented on iOS.

Users who are trying to use `CheckBox` on iOS will get a warning that
> Native component for "AndroidCheckBox" does not exist

We should declare in the document that this component is Android only and use `UnimplementedView` for iOS.

- Use `react-native init` new project
- Apply pull request changes
- Add `<Checkbox />` after welcome text in `App.js`
- Run the app in iOS simulator
Closes #16211

Differential Revision: D6005393

Pulled By: hramos

fbshipit-source-id: 1c9b68b5e1c933496c4d7c2f487f0500264b603a

```

提取一下关键字眼：

>use `UnimplementedView` for iOS.


----

`UnimplementedView`是啥？一脸懵X

查源码呗，以下是`UnimplementedView.js`中的实现 [源码地址](https://github.com/facebook/react-native/blob/26684cf3adf4094eb6c405d345a75bf8c7c0bf88/Libraries/Components/UnimplementedViews/UnimplementedView.js)

``` javascript

class UnimplementedView extends React.Component<$FlowFixMeProps> {
  setNativeProps() {
    // Do nothing.
    // This method is required in order to use this view as a Touchable* child.
    // See ensureComponentIsNative.js for more info
  }

  render() {
    // Workaround require cycle from requireNativeComponent
    const View = require('View');
    return (
      <View style={[styles.unimplementedView, this.props.style]}>
        {this.props.children}
      </View>
    );
  }
}

const styles = StyleSheet.create({
  unimplementedView: __DEV__
    ? {
        alignSelf: 'flex-start',
        borderColor: 'red',
        borderWidth: 1,
      }
    : {},
});

```

可以看到，这个`UnimplementedView`在DEV模式下会提供1个单位的红框（如果我们使用的时候没有指定style）。 除此之外就是个空的View...

所以iOS的同学要使用类似`CheckBox`的组件，就自己实现或者找三方法的实现吧。
其实，`CheckBox`的单选功能，对应在iOS上的组件应该是`Switch`控件喽，多选的话我一般是使用`Button`实现。

见仁见智，网上也有别人实现的`CheckBox`，例如今天被我误解为RN自带库的这家伙（谁让他名字这么官方范）：[https://www.npmjs.com/package/react-native-checkbox](https://www.npmjs.com/package/react-native-checkbox), 不过这Star数也不多哈哈。

之后使用的时候多多尝试吧~


