---
layout:     post
title:      Material Design
subtitle:   Android
date:       2019-08-08
author:     wushiqian
header-img: img/post-bg-temple.jpg
catalog: true
tags:
    - Android
    - UI
---

# Material Design

[参考链接](https://material.io/design/)

## Material Design概述

### 核心思想

所谓Material Design，即**原质化设计**，使用了隐喻的材料设计风格，它不在视觉上模拟特定材料，所以比较容易控制用户的期望。比如在屏幕上画出很像纸质通讯录的通讯录，然而用户期望的纸质通讯录可以翻页，可以折角，可以用笔批注，甚至有纸的触感。而一个不明材料制成的卡片，用户的期望可能只是可以移动而已，而这恰恰是我们想通过卡片表达的隐喻。

### 材质与空间

Material design引入了空间的概念，即：z轴垂直于屏幕，用来表现元素的层叠关系。z轴的值越高，元素离界面底层（水平面）越远，投影越重。这里有一个前提，所有的元素的厚度都是1dp，所有元素都有默认的高度，对它进行操作会抬升它的海拔高度，操作结束后，它应该落回默认海拔高度。同一种元素，同样的操作，抬升的高度是一致的。

### 动画

#### 真实动作

考虑真实世界中的运动规律

#### 响应式交互

- 表层响应

如 水波纹效果

推荐一篇[水波纹源码解读文章](http://www.jianshu.com/p/c23cf6525532)

- 元素响应

如 长按图标浮起

- 径向响应

- 转场动画

在android.transition包下提供关于transitionAnimation的过渡框架

[Android共享元素转场动画](https://www.jianshu.com/p/e87c0086a3ae)

- 细节动画

[lottie-android](https://github.com/airbnb/lottie-android)
![](http://ww4.sinaimg.cn/large/006tNc79ly1g5nov202hlg30m80b4gs9.gif)

### 样式

- 色彩

- 字体

英文字体用Roboto，中文字体用Noto

- 字体排版

12，14，16，20，34号字体

### 图标

### 图像

### 组件

- BottomSheets

- 卡片

- Dialogs

- Menus

- 选择器

- Sliders

- 进度和动态

- Snackbar

- Tab

## Design Support Library

### TextInputLayout

### Snackbar

### FloatActionButton

### NavigationView

### CoordinatorLayout

#### CoordinatorLayout实现ToolBar隐藏效果

#### 结合CollapsingToolbarLayout实现ToolBar折叠效果

#### 自定义Behavior

### SearchView

### Bottom Navigation

### App Shortcuts

### Palette

### BottomAppBar