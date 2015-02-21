---
layout: post  
title: UINavigationController中设置backBarButtonItem显示规则  
category: 技术分享  
tagline: "Supporting tagline"  
tags : [ios,UINavigationController]
---

在UINavigationController中使用pushViewController切换到下一个视图时，UINavigationController会按照以下3条顺序更改导航栏的左侧按钮。

比如我们从A视图到B视图

1. 如果B视图有一个自定义的左侧按钮（leftBarButtonItem），则会显示这个自定义按钮；

2. 如果B没有自定义按钮，但是A视图的backBarButtonItem属性有自定义项，则显示这个自定义项；

3. 如果前2条都没有，则默认显示一个后退按钮，后退按钮的标题是A视图的标题。
<!--break-->
按照这个规则，把UIBarButtonItem *backItem 设置返回按钮的这段代码放在A视图的pushViewController语句之前。这样B视图的后退按钮的标题变成我们设置的了。

{%highlight ios%}
UIBarButtonItem *backItem = [[UIBarButtonItem alloc] initWithTitle:@"back" style:UIBarButtonItemStyleBordered target:nil action:nil];
[self.navigationItem setBackBarButtonItem:backItem];
[backItem release];
[self.navigationController pushViewController:self.bView animated:YES];
{%endhighlight%} 