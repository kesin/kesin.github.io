---
layout: post
title: Haml官网简明教程（翻译）
date: 2017-04-25 15:25:29
excerpt: （2014年3月发表于开源中国博客）这是Haml官网上面的一片很简明的教程，haml使用还是很简单的。原文网址：http://www.haml.info/tutorial.html。
---

最近看的E文文档较多，觉得阅读技术文档的能力有所上升，正好看到一片不算太长的Haml官网的入门教程，就拿来翻译了一下，没用Google，前后弄了40分钟左右  - -，还需要努力啊。。。

####正文：

开始这部分教程之前，需要说明一件事。当你看完本教程，请把你的一个`ERB`文件用`Haml`写出来，尝试一下，写不好删除了也无所谓。如果你不喜欢你写的`haml`文件，那就没必要去保存，但是只要你看完了这个教程，务必尝试转换一个`erb`文档到`haml`。

###开始入门
首先，[获取Haml](http://www.haml.info/download.html)并且[安装Gem](http://www.haml.info/docs/yardoc/file.REFERENCE.html#using_haml)（本教程假设你用的是`ruby on rails`, 在其它的一些框架Haml也是同样的用法），`Haml`是用来替代`ERB`的，这就意味着你的`app/views`文件夹下面的所有文件的后缀名都应该改成`haml`的文件名。

```
app/views/account/login.html.erb → app/views/account/login.html.haml
```

###如何去转换

让我们开始转换一些基本的ERB样式

```
**<%= item.title %>**    #ERB

%strong= item.title    #Haml
```

在`Haml`中，我们用百分号`%`去写一个标签，百分号紧接着就是标签的名称。`%strong,%div,%body,%html`等任何你想用的标签。然后紧接着标签的是`=`符号，这个符号告诉`Haml`正确的解释`Ruby`代码并且输出返回值作为这个标签的内容。不像`ERB`，`Haml`会自动检测返回值并且正确的格式化标签。
增加标签属性

简单的标签看起来没有问题，但是如果为标签加上属性呢？
```
**Hello, World!**    #HTML

%strong{:class => "code", :id => "message"} Hello, World!      #Haml
```
这些属性其实就是标准的`Ruby hash`表，`class`的属性是`code`，`id`的属性是`message`。注意到在这个例子中我们没有用等号 `=`  ，所以，`Hello World`被解释成正常的字符，而不是`ruby`代码。
有这么一个简单的方法去定义`Haml`中的标签，由于`class`和`id`是`css`中比较相同的属性并且被大多数设计人员和开发人员所熟悉的，我们可以用简单的符号去描述这个标签
```
%strong.code#message Hello, World!     #Haml
```
而且不止这些，`div`这个标签使用的太普遍了，所以你可以只写`tag`的属性，让它默认为 `%div`，如下
```
.content Hello, World!     #Haml

<div class='content'>Hello, World!</div>      #Html
```
###使用进阶

现在来点比较复杂的怎么样？

`ERB`代码：
```
<div class='item' id='item<%= item.id %>'>
  <%= item.body %>
</div>
```
相当基础，这可能是局部代码的一部分或是什么东东，把它转换到`Haml`吧
`Haml`代码
```
.item{:id => "item#{item.id}"}= item.body
```
干的很不错，现在把下面白框里面的ERB代码转换成Haml吧
`ERB`代码
```
<div id='content'>
  <div class='left column'>

## Welcome to our site!

<%= print_information %>

  </div>
  <div class="right column">
    <%= render :partial => "sidebar" %>
  </div>
</div>
```
转换后：`Haml`
```
#content
  .left.column
    %h2 Welcome to our site!
    %p= print_information
  .right.column
    = render :partial => "sidebar"
```
看看这代码，难道不会让你露出一丝微笑？
还有很多要学的，强烈推荐你翻阅我们的[手册](http://www.haml.info/docs/yardoc/file.REFERENCE.html)，手册里面有很多我们增加到Haml的精彩的小技巧，能让开发网站变得更加有趣。