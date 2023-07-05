---
title: CTFd 管理面板题目提示界面显示错误修复
description: 难以理解的提示创建界面，全是前端的问题。。。
categories:
- CTFd
tags:
- CTFd
- CTFd-Challenge
- Vue
date: 2023-07-05
photos: /ctfd/2023/07/05/CTFd-Challenge-Hint-Fix/image-20230705160621887.png
---
CTFd的题目允许出题人增加提示，但是增加提示窗口最下面的地方却比较奇怪。。。

![e84d8fc3b3ee3062a83866d89e3db27](e84d8fc3b3ee3062a83866d89e3db27.png)

![e2c4a7dfbe6e1f3f6ba6729bd165c66](e2c4a7dfbe6e1f3f6ba6729bd165c66.png)

需求0-42，ID static，这都是些什么乱七八糟的？

![03c4933341826ffdbd737960669b00c](03c4933341826ffdbd737960669b00c.png)

![5c4788a462a3dd72a010c43a98e3a14](5c4788a462a3dd72a010c43a98e3a14.png)

查看源代码可知，原来本应该是ID的地方写成了Type，而神秘的0-42则是提示的花销和提示的ID，有点逆天只能说。

修改成正确的就可以了，改完的效果如下

![image-20230705160553765](image-20230705160553765.png)

![image-20230705160621887](image-20230705160621887.png)

这样就正常了。

PS：我们Scr1w战队二次开发的CTFd整合版地址：https://github.com/dlut-sss/CTFD-Public

