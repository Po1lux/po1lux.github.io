---
title: test
date: 2018-09-22 18:08:59
tags:
---
![](test/1.jpg)
> 引用


#标题1
##标题2
###标题3
####标题4
#####标题5
*斜体*
**加黑1**

- 列表1
- 列表2

**加黑**

- **列表加黑**

``` python
@requires_authorization
def somefunc(param1='', param2=0):
    '''A docstring'''
    if param1 > param2: # interesting
        print 'Greater'
    return (param2 - param1 + 1) or None
class SomeClass:
    pass
>>> message = '''interpreter
... prompt'''
```

短代码`class SomeClass:`


使用 `- [ ]` 和 `- [x]` 语法可以创建复选框，实现 todo-list 等功能。例如：

- [x] 已完成事项
- [ ] 待办事项1
- [ ] 待办事项2