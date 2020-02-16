### Module

```shell
>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> import sys
>>> dir()      
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'sys']
>>> sys.modules['os'] 
<module 'os' from 'D:\\mygarbagecode\\playground\\Python-3.7.0\\lib\\os.py'>
>>> sys.modules['module2']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'module2'
>>> import module2
module 2
>>> sys.modules['module2']
<module 'module2' from 'D:\\mygarbagecode\\playground\\Python-3.7.0\\PCbuild\\win32\\test\\module2.py'>
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'module2', 'sys']
```

当前命名空间内的 module2 和 sys 的 module2 是同一个对象，并且 sys 的 module2 是由于在 `import module2` 引入的，因为在 import 之前 sys 的 modules 对象并没有 'module2' （ KeyError ） 。

```shell
>>> id(module2) 
55301160
>>> id(sys.modules['module2']) 
55301160
```

##### PyModuleDef

PyModuleDef 的定义如下：

```c
typedef struct PyModuleDef{
  PyModuleDef_Base m_base;
  const char* m_name;
  const char* m_doc;
  Py_ssize_t m_size;
  PyMethodDef *m_methods;
  struct PyModuleDef_Slot* m_slots;
  traverseproc m_traverse;
  inquiry m_clear;
  freefunc m_free;
} PyModuleDef;
```

sysmodule 定义如下：

```c
static struct PyModuleDef sysmodule = {
    PyModuleDef_HEAD_INIT,
    "sys",
    sys_doc,
    -1, /* multiple "initialization" just copies the module dict. */
    sys_methods,
    NULL,
    NULL,
    NULL,
    NULL
};
```

sys_methods 就是 sys 这个模块提供的功能。

##### IMPORT_NAME

Python 代码和字节码如下：

```python
import module2
```

```shell
  1           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (None)
              4 IMPORT_NAME              0 (module2)
              6 STORE_NAME               0 (module2)
```

IMPORT_NAME 字节码对应代码如下：

```c
TARGET(IMPORT_NAME) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *fromlist = POP();
    PyObject *level = TOP();
    PyObject *res;
    res = import_name(f, name, fromlist, level);
    Py_DECREF(level);
    Py_DECREF(fromlist);
    SET_TOP(res);
    if (res == NULL)
        goto error;
    DISPATCH();
}
```

执行 IMPORT_NAME 后的结果其实就是把栈顶的两个参数 POP 后替换为 module2 模块，最终通过 STORE_NAME 引入 locals 命名空间。

这里的 name 就是 'module2' ，from_list 为 None ，level 为 0 。这三个参数会连同 当前栈帧 f 调用 import_name 。

```c
static PyObject *
import_name(PyFrameObject *f, PyObject *name, PyObject *fromlist, PyObject *level)
{
    _Py_IDENTIFIER(__import__);
    PyObject *import_func, *res;
    PyObject* stack[5];

    import_func = _PyDict_GetItemId(f->f_builtins, &PyId___import__);
    if (import_func == NULL) {
        PyErr_SetString(PyExc_ImportError, "__import__ not found");
        return NULL;
    }

    /* Fast path for not overloaded __import__. */
    ...

    Py_INCREF(import_func);

    stack[0] = name;
    stack[1] = f->f_globals;
    stack[2] = f->f_locals == NULL ? Py_None : f->f_locals;
    stack[3] = fromlist;
    stack[4] = level;
    res = _PyObject_FastCall(import_func, stack, 5);
    Py_DECREF(import_func);
    return res;
}
```

关于 `__import__` 及其参数的相关信息可以在[这里]( https://docs.python.org/3.7/library/functions.html#__import__ )查看。这里搬运一下官方的两个例子：

>For example, the statement `import spam` results in bytecode resembling the following code:
>
>```python
>spam = __import__('spam', globals(), locals(), [], 0)
>```
>
>The statement `import spam.ham` results in this call:
>
>```python
>spam = __import__('spam.ham', globals(), locals(), [], 0)
>```

如果用户没有自行给出 `__import__` 的实现，那么最终其实会调用 `builtin___import__` ：

```c
// bltinmodule.c
static PyMethodDef builtin_methods[] = {
    ...
    {"__import__",      (PyCFunction)builtin___import__, METH_VARARGS | METH_KEYWORDS, import_doc},
    ...
}
```

##### from import

众所周知， Python 提供了 from import 语法：

```shell
>>> from module2 import a
module 2
>>> from module2 import b
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'a', 'b']
>>> import sys
>>> sys.modules['module2']
<module 'module2' from 'D:\\mygarbagecode\\playground\\Python-3.7.0\\PCbuild\\win32\\test\\module2.py'>
>>> sys.modules['module2.a'] 
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'module2.a'
>>> sys.modules['a']         
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'a'
```

可以看到， module2 这个包在第二次被使用时并没有打印 'module 2' ，这是因为它已经被缓存（在 sys.modules 中）了。另外，from mod import sth 对 sys.modules 的影响其实与前面的 import mod 是一样的，最终都会在 sys.modules 内放入 ('module2', `<mod>`) 键值对，区别只是当前目录下引入的名字不同而已。 

给出以下 Python 代码：

```python
from module2 import a, b
```

```shell
  2           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (('a', 'b'))
              4 IMPORT_NAME              0 (module2)
              6 IMPORT_FROM              1 (a)
              8 STORE_NAME               1 (a)
             10 IMPORT_FROM              2 (b)
             12 STORE_NAME               2 (b)
             14 POP_TOP
             16 LOAD_CONST               2 (None)
             18 RETURN_VALUE
```

来到 IMPORT_FROM 之前的三行情诗我们已经见过，如果是 import module2 这里再接一个 STORE_NAME 就已经结束了。

IMPORT_FROM 对应代码如下：

```c
TARGET(IMPORT_FROM) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *from = TOP();
    PyObject *res;
    res = import_from(from, name);
    PUSH(res);
    if (res == NULL)
        goto error;
    DISPATCH();
}
```

在每次 IMPORT_FROM 时栈顶都是 module2 这个模块，执行一次 IMPORT_FROM 和 STORE_NAME 就引入了一个名字。

##### 名字的删除

引入一个名字后可以通过 del name 进行 undef ：

```shell
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'a', 'b', 'sys']
>>> del a
>>> del b
>>> dir() 
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'sys']
>>> sys.modules['module2'] 
<module 'module2' from 'D:\\mygarbagecode\\playground\\Python-3.7.0\\PCbuild\\win32\\test\\module2.py'>
>>>
```

可以看到，虽然 dir() 的输出中已经没有了名字 a 和 b ，但在 sys.modules 依旧存在 module2 ，只是 Python 向我们隐藏了细节。

##### builtin__import

builtin__import 最终会调用 PyImport_ImportModuleLevelObject ：

```c
PyObject *
PyImport_ImportModuleLevelObject(PyObject *name, PyObject *globals,
                                 PyObject *locals, PyObject *fromlist,
                                 int level)
{
    ...

    mod = PyImport_GetModule(abs_name);
    if (mod != NULL && mod != Py_None) {
        ...
    }
    else {
        Py_XDECREF(mod);
        mod = import_find_and_load(abs_name);
        if (mod == NULL) {
            goto error;
        }
    }

    ...
        
    if (!has_from) {
        Py_ssize_t len = PyUnicode_GET_LENGTH(name);
        if (level == 0 || len > 0) {
            Py_ssize_t dot;

            dot = PyUnicode_FindChar(name, '.', 0, len, 1);
            if (dot == -2) {
                goto error;
            }

            if (dot == -1) {
                /* No dot in module name, simple exit */
                final_mod = mod;
                Py_INCREF(mod);
                goto error;
            }

            if (level == 0) {
                PyObject *front = PyUnicode_Substring(name, 0, dot);
                if (front == NULL) {
                    goto error;
                }
                // 递归
                final_mod = PyImport_ImportModuleLevelObject(front, NULL, NULL, NULL, 0);
                Py_DECREF(front);
            }
            else {
                Py_ssize_t cut_off = len - dot;
                Py_ssize_t abs_name_len = PyUnicode_GET_LENGTH(abs_name);
                PyObject *to_return = PyUnicode_Substring(abs_name, 0,
                                                        abs_name_len - cut_off);
                if (to_return == NULL) {
                    goto error;
                }

                final_mod = PyImport_GetModule(to_return);
                Py_DECREF(to_return);
                if (final_mod == NULL) {
                    PyErr_Format(PyExc_KeyError,
                                 "%R not in sys.modules as expected",
                                 to_return);
                    goto error;
                }
            }
        }
        else {
            final_mod = mod;
            Py_INCREF(mod);
        }
    }
    else {
        final_mod = _PyObject_CallMethodIdObjArgs(interp->importlib,
                                                  &PyId__handle_fromlist, mod,
                                                  fromlist, interp->import_func,
                                                  NULL);
    }

  error:
    Py_XDECREF(abs_name);
    Py_XDECREF(mod);
    Py_XDECREF(package);
    if (final_mod == NULL)
        remove_importlib_frames();
    return final_mod;
}
```

由 abs_name 和 level 拿到 abs_name 后，虚拟机会尝试在 sys.modules 中获取对应的 module ，失败则调用 import_find_and_load 。

先来没有 from_list 参数时的情况，也就是类似 import x.y.z ，如果没有 dot （ import x ），会直接跳转到 error 并将 final_mod 返回（这里的 error label 处理的情况应该不一定都是 error）。如果存在 dot ，那么其实就是 import package 中的某个 module ，这种情况下其实最终返回的是整个 package ，注意这里我们截取的是第一个 dot 之前的 front 。

当存在 from_list 参数时，则会调用 _PyObject_CallMethodIdObjArgs （待补充）。