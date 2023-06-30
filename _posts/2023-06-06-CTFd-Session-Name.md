---
title: CTFd Session污染处理
description: 众所周知，有些题目是需要cookie或session进行解题的，但是你可能拿到的session被平台污染。
categories:
- CTFd
tags:
- CTFd
- Session
date: 2023-06-01
---
这里简短说一下原理，CTFd在不配置的情况下使用的session名是默认的，会跟其他题目内部的session名重复，如果是通过direct frp将平台和docker靶机部署在同一host下，那么会产生Cookie污染。
解决办法也很简单，修改CTFd Session名。
在/CTFd/__init__.py里

```python
def create_app(config="CTFd.config.Config"):
    app = CTFdFlask(__name__)
    app.config['SESSION_COOKIE_NAME'] = 'Scr1wCTFDSessionId' # 改成自己的新Session名就行
    with app.app_context():
        app.config.from_object(config)
        ...
```
