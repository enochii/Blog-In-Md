### 对象创建

[TOC]

字节码的解析逻辑事实上大部分都是在`_PyEval_EvalFrameDefault`这个函数中实现的，无特殊说明，以下代码均摘自`_PyEval_EvalFrameDefault`。

#### 简单对象创建

```python
i = 1

s = 'xixixi'

l = []

d = {}
```

这里指的简单对象是整数，字符串及空列表和空字典，使用`dis`工具得字节码如下：

```shell
  1           0 LOAD_CONST               0 (1)
              2 STORE_NAME               0 (i)

  3           4 LOAD_CONST               1 ('xixixi')
              6 STORE_NAME               1 (s)

  5           8 BUILD_LIST               0
             10 STORE_NAME               2 (l)

  7          12 BUILD_MAP                0
             14 STORE_NAME               3 (d)
             16 LOAD_CONST               2 (None)
             18 RETURN_VALUE
```

可以看到，整数和字符串的创建都是执行了两个指令：

```shell
			  0 LOAD_CONST               0 (1)
              2 STORE_NAME               0 (i)
```

我们可以到`_PyEval_EvalFrameDefault`中看看这两条指令的实现，首先是

```c
TARGET(LOAD_CONST) {
    PyObject *value = GETITEM(consts, oparg);
    Py_INCREF(value);
    PUSH(value);
    FAST_DISPATCH();
}
```

`LOAD_CONST`执行的动作实际上就是从`consts`常量表中取出`oparg`(0)对应的常量(1)，接着增加其引用计数并压栈，执行下一条指令。

此外，`opcode`和`oparg`是通过`NEXT_OPARG`宏提取得到的：

```c
#define NEXTOPARG()  do { \
        _Py_CODEUNIT word = *next_instr; \
        opcode = _Py_OPCODE(word); \
        oparg = _Py_OPARG(word); \
        next_instr++; \
    } while (0)
```

接着是`STORE_NAME`：

```c
TARGET(STORE_NAME) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *v = POP();
    PyObject *ns = f->f_locals;
    int err;
    if (ns == NULL) {
        PyErr_Format(PyExc_SystemError,
                     "no locals found when storing %R", name);
        Py_DECREF(v);
        goto error;
    }
    if (PyDict_CheckExact(ns))
        err = PyDict_SetItem(ns, name, v);
    else
        err = PyObject_SetItem(ns, name, v);
    Py_DECREF(v);
    if (err != 0)
        goto error;
    DISPATCH();
}
```

先是在符号表中根据`oparg`拿出对应的`name`，把该键值对存储进当前栈帧`f`的`f_locals`命名空间中。

我们可以看到，**执行下一条指令**事实上是有两种手法的：

```c
#define DISPATCH() continue
#define FAST_DISPATCH() goto fast_next_opcode
```

`DISPATCH`比`FAST_DISPATCH`多做了一些工作：

```c
for (;;) {
        assert(stack_pointer >= f->f_valuestack); /* else underflow */
        assert(STACK_LEVEL() <= co->co_stacksize);  /* else overflow */
        assert(!PyErr_Occurred());

        /* Do periodic things.  Doing this every time through the loop would add too much overhead, so we do it only every Nth instruction. We also do it if ``pendingcalls_to_do'' is set, i.e. when an asynchronous event needs attention (e.g. a signal handler or async I/O handler); see Py_AddPendingCall() and Py_MakePendingCalls() above. */
if(_Py_atomic_load_relaxed(&_PyRuntime.ceval.eval_breaker))		{
        ...
    }

    fast_next_opcode:
    ...
```

对于空列表和空字典，执行的操作和整数、字符串其实差不多。

最后两行的

```shell
            16 LOAD_CONST               2 (None)
            18 RETURN_VALUE
```

> Python执行一段Code Block后，一定要返回一些值，这里入栈的事实上是一个None值

#### 复杂对象创建

然后来看看非空列表和字典的初始化：

```python
d = {"1":1, "2":2}
l = [1,2]
```

字节码如下：

```shell
  1           0 LOAD_CONST               0 (1)
              2 LOAD_CONST               1 (2)
              4 LOAD_CONST               2 (('1', '2'))
              6 BUILD_CONST_KEY_MAP      2
              8 STORE_NAME               0 (d)

  2          10 LOAD_CONST               0 (1)
             12 LOAD_CONST               1 (2)
             14 BUILD_LIST               2
             16 STORE_NAME               1 (l)
             18 LOAD_CONST               3 (None)
             20 RETURN_VALUE
```

先看看`d = {"1":1, "2":2}`生成的字节码，首先是把`values`都压入了运行时栈，

这里的所有的`key`值事实上是做成了一个tuple，下面是`BUILD_CONST_KEY_MAP`的实现：

```c
TARGET(BUILD_CONST_KEY_MAP) {
            Py_ssize_t i;
            PyObject *map;
            PyObject *keys = TOP();
            if (!PyTuple_CheckExact(keys) ||
                PyTuple_GET_SIZE(keys) != (Py_ssize_t)oparg)			{
                PyErr_SetString(PyExc_SystemError,
                                "bad BUILD_CONST_KEY_MAP keys argument");
                goto error;
            }
            map = _PyDict_NewPresized((Py_ssize_t)oparg);
            if (map == NULL) {
                goto error;
            }
            for (i = oparg; i > 0; i--) {
                int err;
                PyObject *key = PyTuple_GET_ITEM(keys, oparg - i);
                PyObject *value = PEEK(i + 1);
                err = PyDict_SetItem(map, key, value);
                if (err != 0) {
                    Py_DECREF(map);
                    goto error;
                }
            }

            Py_DECREF(POP());
            while (oparg--) {
                Py_DECREF(POP());
            }
            PUSH(map);
            DISPATCH();
        }
```

首先取出栈顶的`values`元组：

```c
	PyObject *keys = TOP();
	if (!PyTuple_CheckExact(keys) ||
            PyTuple_GET_SIZE(keys) != (Py_ssize_t)oparg)
    {
        PyErr_SetString(PyExc_SystemError,
                          "bad BUILD_CONST_KEY_MAP keys argument");
        goto error;
    }
```

其中`oparg`为BUILD_MAP的参数（也就是初始化一个字典时，键值对的数量），通过循环取出对应的键值对，调用`PyDict_SetItem`将其加入新建的字典。其中`PEEK`宏用于取出value值：

```C
#define PEEK(n)           (stack_pointer[-(n)])
```

`BUILD_LIST`逻辑类似且相对简单，在此不再细究，以下为对应代码：

```c
TARGET(BUILD_LIST) {
      PyObject *list =  PyList_New(oparg);
      if (list == NULL)
            goto error;
      while (--oparg >= 0) {
            PyObject *item = POP();
            PyList_SET_ITEM(list, oparg, item);
       }
       PUSH(list);
       DISPATCH();
}
```

