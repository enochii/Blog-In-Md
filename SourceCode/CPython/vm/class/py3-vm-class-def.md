### 自定义class

给出下列简单类定义：

```python
class A(object):
    name = 'Python'
    def __init__(self):
        print('A::init')
        
    def fun(self):
        print("A::fun()")
        
    def gun(self, aVal):
        self.val = aVal
        print(self.val)
```

用 dis 工具得到字节码如下：

```shell
  1           0 LOAD_BUILD_CLASS
              2 LOAD_CONST               0 (<code object A at 0x033202C8, file "test/class_test2.py", line 1>)
              4 LOAD_CONST               1 ('A')
              6 MAKE_FUNCTION            0
              8 LOAD_CONST               1 ('A')
             10 LOAD_NAME                0 (object)
             12 CALL_FUNCTION            3
             14 STORE_NAME               1 (A)


Disassembly of <code object A at 0x033202C8, file "test/class_test2.py", line 1>:
  1           0 LOAD_NAME                0 (__name__)
              2 STORE_NAME               1 (__module__)
              4 LOAD_CONST               0 ('A')
              6 STORE_NAME               2 (__qualname__)

  2           8 LOAD_CONST               1 ('Python')
             10 STORE_NAME               3 (name)

  3          12 LOAD_CONST               2 (<code object __init__ at 0x03320728, file "test/class_test2.py", line 3>)
             14 LOAD_CONST               3 ('A.__init__')
             16 MAKE_FUNCTION            0
             18 STORE_NAME               4 (__init__)

  6          20 LOAD_CONST               4 (<code object fun at 0x03320258, file "test/class_test2.py", line 6>)
             22 LOAD_CONST               5 ('A.fun')
             24 MAKE_FUNCTION            0
             26 STORE_NAME               5 (fun)

  9          28 LOAD_CONST               6 (<code object gun at 0x03320B88, file "test/class_test2.py", line 9>)
             30 LOAD_CONST               7 ('A.gun')
             32 MAKE_FUNCTION            0
             34 STORE_NAME               6 (gun)
             36 LOAD_CONST               8 (None)
             38 RETURN_VALUE

Disassembly of <code object __init__ at 0x03320728, file "test/class_test2.py", line 3>:
  4           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('A::init')
              4 CALL_FUNCTION            1
              6 POP_TOP
              8 LOAD_CONST               0 (None)
             10 RETURN_VALUE

Disassembly of <code object fun at 0x03320258, file "test/class_test2.py", line 6>:
  7           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('A::fun()')
              4 CALL_FUNCTION            1
              6 POP_TOP
              8 LOAD_CONST               0 (None)
             10 RETURN_VALUE

Disassembly of <code object gun at 0x03320B88, file "test/class_test2.py", line 9>:
 10           0 LOAD_FAST                1 (aVal)
              2 LOAD_FAST                0 (self)
              4 STORE_ATTR               0 (val)

 11           6 LOAD_GLOBAL              1 (print)
              8 LOAD_FAST                0 (self)
             10 LOAD_ATTR                0 (val)
             12 CALL_FUNCTION            1
             14 POP_TOP
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE
```

首先看位于前排的字节码：

```shell
  1           0 LOAD_BUILD_CLASS
              2 LOAD_CONST               0 (<code object A at 0x033202C8, file "test/class_test2.py", line 1>)
              4 LOAD_CONST               1 ('A')
              6 MAKE_FUNCTION            0
              8 LOAD_CONST               1 ('A')
             10 LOAD_NAME                0 (object)
             12 CALL_FUNCTION            3
             14 STORE_NAME               1 (A)
```

LOAD_BUILD_CLASS，再陌生不过的字节码，可以看[这里]( https://docs.python.org/3.7/library/dis.html?highlight=load_build_class#opcode-LOAD_BUILD_CLASS )：

>```
>LOAD_BUILD_CLASS
>```
>
>Pushes `builtins.__build_class__()` onto the stack. It is later called by [`CALL_FUNCTION`](https://docs.python.org/3.7/library/dis.html?highlight=load_build_class#opcode-CALL_FUNCTION) to construct a class.

接下来的三句字节码创建了一个函数对象并压入栈中，其中函数对象的 code object 是类 A 的定义：

```shell
              2 LOAD_CONST               0 (<code object A at 0x033202C8, file "test/class_test2.py", line 1>)
              4 LOAD_CONST               1 ('A')
              6 MAKE_FUNCTION            0
```

接着是将字符串 'A' 和 object 压栈，接着执行：

```shell
             12 CALL_FUNCTION            3
```

在执行 CALL_FUNCTION 字节码之前，栈内存示意图如下：

<img src="py3-vm-class-def.assets/image-20200209232416992.png" alt="image-20200209232416992" style="zoom:50%;" />

这里的 CALL_FUNCTION 的 oparg 是3，很明显 CALL_FUNCTION 调用的函数是所谓的 `builtins.__build_class__()`。为了证实这一点，我们分别在 LOAD_BUILD_CLASS 和 call_function 处添加了打印信息：

```c
TARGET(LOAD_BUILD_CLASS) {
    _Py_IDENTIFIER(__build_class__);

    PyObject *bc;
    ...
    PUSH(bc);
    printf("[LOAD_BUILD_CLASS] bc = %p\n", bc);
    ...
}
```



```c
y_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{
    /*
     * 栈是向高处（可查看BASIC_PUSH 定义）增长的 (在压栈时 先压入 func，后压入参数)
     * 这里 oparg 是 位置参数 和 键参数 个数之和
    */
    PyObject **pfunc = (*pp_stack) - oparg - 1;
    PyObject *func = *pfunc;
    printf("[call_function] func = %p \n", func);
    ...
}
```

下列为对应的输出，可以看到 call_function 中的 func 正是在 LOAD_BUILD_CLASS 压入的 bc。

<img src="py3-vm-class-def.assets/image-20200209233136885.png" alt="image-20200209233136885" style="zoom: 67%;" />

前面我们说到，call_function 会在三种不同的情况下被调用：

```c
	//call_function 内部的三个分支
	if (PyCFunction_Check(func)) {
        ...
    }
    else if (Py_TYPE(func) == &PyMethodDescr_Type) {
        ...
    }
    else {
        ...
    }
```

在 LOAD_BUILD_CLASS 走的其实是第一个分支，因为 func 包装的内层其实是一个 CFunction。也就是这里：

```c
	if (PyCFunction_Check(func)) {
        PyThreadState *tstate = PyThreadState_GET();
        C_TRACE(x, _PyCFunction_FastCallKeywords(func, stack, nargs, kwnames));
    }
```

进入 _PyCFunction_FastCallKeywords 内部你会发现 func 其实是一个 PyCFunctionObject：

```c
typedef struct {
    PyObject_HEAD
    PyMethodDef *m_ml; /* Description of the C function to call */
    PyObject    *m_self; /* Passed as 'self' arg to the C func, can be NULL */
    PyObject    *m_module; /* The __module__ attribute, can be anything */
    PyObject    *m_weakreflist; /* List of weak references */
} PyCFunctionObject;
```

真实的 CFunction 被包在 `PyMethodDef *m_ml` 这个域中：

```c
struct PyMethodDef {
    const char  *ml_name;   /* The name of the built-in function/method */
    PyCFunction ml_meth;    /* The C function that implements it */
    int         ml_flags;   /* Combination of METH_xxx flags, which mostly
                               describe the args expected by the C func */
    const char  *ml_doc;    /* The __doc__ attribute, or NULL */
};
typedef struct PyMethodDef PyMethodDef;
```

所以我们通过如下语句打印该函数指针的值：

```c
printf("[call_function] cfunction = %p\n", ((PyCFunctionObject*)func)->m_ml->ml_meth);
```

你会发现其实这个 CFunction 就是 `builtin___build_class__` ：

```c
/* AC: cannot convert yet, waiting for *args support */
static PyObject *
builtin___build_class__(PyObject *self, PyObject *const *args, Py_ssize_t nargs,
                        PyObject *kwnames)
{
    printf("[builtin___build_class__] builtin___build_class__ = %p\n", builtin___build_class__);
    ...
}
```

<img src="py3-vm-class-def.assets/image-20200209234146293.png" alt="image-20200209234146293" style="zoom:67%;" />

这样一来自定义 class 前排的字节码就初步了解了逻辑。注意到在 `builtin___build_class__` 中：

```c
cell = PyEval_EvalCodeEx(PyFunction_GET_CODE(func), PyFunction_GET_GLOBALS(func), ns,
                             NULL, 0, NULL, 0, NULL, 0, NULL,
                             PyFunction_GET_CLOSURE(func));
```

这里的 func 实际上是执行 MAKE_FUNCTION 放在栈中的函数对象，我们说过它的 code object 对应的即是自定义 class 的 code object。也就是：

```shell
Disassembly of <code object A at 0x033202C8, file "test/class_test2.py", line 1>:
  1           0 LOAD_NAME                0 (__name__)
              2 STORE_NAME               1 (__module__)
              4 LOAD_CONST               0 ('A')
              6 STORE_NAME               2 (__qualname__)

  2           8 LOAD_CONST               1 ('Python')
             10 STORE_NAME               3 (name)

  3          12 LOAD_CONST               2 (<code object __init__ at 0x03320728, file "test/class_test2.py", line 3>)
             14 LOAD_CONST               3 ('A.__init__')
             16 MAKE_FUNCTION            0
             18 STORE_NAME               4 (__init__)

  6          20 LOAD_CONST               4 (<code object fun at 0x03320258, file "test/class_test2.py", line 6>)
             22 LOAD_CONST               5 ('A.fun')
             24 MAKE_FUNCTION            0
             26 STORE_NAME               5 (fun)

  9          28 LOAD_CONST               6 (<code object gun at 0x03320B88, file "test/class_test2.py", line 9>)
             30 LOAD_CONST               7 ('A.gun')
             32 MAKE_FUNCTION            0
             34 STORE_NAME               6 (gun)
             36 LOAD_CONST               8 (None)
             38 RETURN_VALUE
```

前四句字节码把 `__name__`对应的值绑定到了 `__module__` ，将 'A' 绑定到了 `__qualname__`。接着就是实际的 python 代码逻辑，包括一个赋值语句和三个函数定义（把 code object 和函数名绑定起来）。

类 A 中比较有趣的函数定义当属 gun()：

```python
    def gun(self, aVal):
        self.val = aVal
        print(self.val)
```



```shell
Disassembly of <code object gun at 0x03320B88, file "test/class_test2.py", line 9>:
 10           0 LOAD_FAST                1 (aVal)
              2 LOAD_FAST                0 (self)
              4 STORE_ATTR               0 (val)

 11           6 LOAD_GLOBAL              1 (print)
              8 LOAD_FAST                0 (self)
             10 LOAD_ATTR                0 (val)
             12 CALL_FUNCTION            1
             14 POP_TOP
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE
```

从字节码 LOAD_FAST 可以看出，实参和对象指针 self 都是放在当前的 f_localsplus 中的。没看过的字节码就是 STORE_ATTR 和 LOAD_ATTR。比如 LOAD_ATTR：

```c
TARGET(LOAD_ATTR) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = TOP();
    PyObject *res = PyObject_GetAttr(owner, name);
    Py_DECREF(owner);
    SET_TOP(res);
    if (res == NULL)
        goto error;
    DISPATCH();
}
```

这里的 owner 其实就是栈顶的 self 啦。









函数调用产生的字节码如下：

```shell
 13          16 LOAD_NAME                1 (A)
             18 CALL_FUNCTION            0
             20 STORE_NAME               2 (a)

 14          22 LOAD_NAME                2 (a)
             24 LOAD_METHOD              3 (fun)
             26 CALL_METHOD              0
             28 POP_TOP

 15          30 LOAD_NAME                2 (a)
             32 LOAD_METHOD              4 (gun)
             34 LOAD_CONST               2 (10)
             36 CALL_METHOD              1
             38 POP_TOP
             40 LOAD_CONST               3 (None)
             42 RETURN_VALUE
```

