---
title: 这样理解 Python 闭包
date: 2016-08-23 21:02:36
tags: [Python, closure]
categories: [Tech Notes]

---

看了 [12 步轻松搞定 python 装饰器](http://python.jobbole.com/81683/) 里的描述后，对『closure 是上下文』有了些形象的理解，一些应用了 closure 的实践可以参考文章里的一些例子，例如函数参数定制生成，装饰器功能等




```py

g_var = 20

def outer():
    loc_var = 30
    def inner():
        print('I can access g_var: %s, loc_var: %s' % (g_var, loc_var))
    return inner

in_fun = outer()

# 内层使用到的[外层函数的本地变量]会加入到 func 对象的 __closure__ tuple 里面
print(in_fun.__closure__)

# 全局变量不包含在 closure tuple，并不是副本
g_var = 5000

# 效果就是 in_fun 本身函数内的代码，加上 __closure__ 里的上下文副本实现了 closure 功能
in_fun()

# 也就是说 __closure__ tuple 包含一个 cell 元素，cell 装有外层的 loc_var，值是 30
print('There is %d cell inside' % len(in_fun.__closure__))
print('This cell contain %d' % in_fun.__closure__[0].cell_contents)


```



