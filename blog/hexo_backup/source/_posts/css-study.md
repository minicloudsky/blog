---
title: css学习
date: 2021-06-14 19:23:09
tags:
---


## CSS 选择器用于“查找”（或选取）要设置样式的 HTML 元素。

我们可以将 CSS 选择器分为五类：
简单选择器（根据名称、id、类来选取元素）
组合器选择器（根据它们之间的特定关系来选取元素）
伪类选择器（根据特定状态选取元素）
伪元素选择器（选取元素的一部分并设置其样式）
属性选择器（根据属性或属性值来选取元素）
此页会讲解最基本的 CSS 选择器。
### css元素选择器
元素选择器根据元素名称来选择html元素
页面所有p标签居中对齐，文本颜色为红色
```css
p {
    text-align: center;
    color: red;
}
```
### id选择器
id选择器使用html元素的id属性来选择特定元素
元素id在页面中是唯一的，因此id选择器用于选择一个唯一的元素
要选择具有特定id的元素，请写一个井号(#),后面跟该元素的id
```css
 #para1 {
     text-align: center;
     color: red;
 }
```

<!-- more -->
### 类选择器
类选择器选择有特定 class 属性的html元素
如需选择拥有特定 class 的元素，请写一个句点(.)字符，后面跟类名。
所有带有 class="centter"的 html 元素将为红色且居中对齐
```css
.center {
    text-align: centter;
    color: red;
}
```
### html元素也可以引用多个类
在这个例子中，<p> 元素将根据 class="center" 和 class="large" 进行样式设置：
```html
<p class="center large">这个段落引用两个类。</p>
```
### 通用选择器
通用选择器(*) 选择页面上的所有html元素
下面的css规则会影响页面上的每个html元素
### 选择器分组
h1,h2,p元素具有同样的样式定义
```css
css
 *{
     text-align: center;
     color: blue;
 }
```css
h1 {
  text-align: center;
  color: red;
}

h2 {
  text-align: center;
  color: red;
}

p {
  text-align: center;
  color: red;
}
```
对上述代码中的选择器进行分组
```css
h1,h2,p{
  text-align: center;
  color: red;
}
```