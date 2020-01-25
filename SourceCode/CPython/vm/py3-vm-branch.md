### 分支

```python
a = 1

if a > 10:
    print("a > 10")
else:
    print("xixixi")
```

字节码如下：

```shell
  1           0 LOAD_CONST               0 (1) 
              2 STORE_NAME               0 (a) 

  3           4 LOAD_NAME                0 (a) 
              6 LOAD_CONST               1 (10)
              8 COMPARE_OP               4 (>)
             10 POP_JUMP_IF_FALSE       22

  4          12 LOAD_NAME                1 (print)
             14 LOAD_CONST               2 ('a > 10')
             16 CALL_FUNCTION            1
             18 POP_TOP
             20 JUMP_FORWARD             8 (to 30)

  6     >>   22 LOAD_NAME                1 (print)
             24 LOAD_CONST               3 ('xixixi')
             26 CALL_FUNCTION            1
             28 POP_TOP
        >>   30 LOAD_CONST               4 (None)
             32 RETURN_VALUE
```

##### 参数压栈

前两句字节码（由`a = 1`生成）与之前相同，在此不再赘述。而`LOAD_NAME`事实上是`STORE_NAME`的逆动作：

```c
TARGET(LOAD_NAME) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *locals = f->f_locals;
    PyObject *v;
    if (locals == NULL) {
        PyErr_Format(PyExc_SystemError,
                     "no locals when loading %R", name);
        goto error;
    }
    /*先从local寻找该名字*/
    if (PyDict_CheckExact(locals)) {
        v = PyDict_GetItem(locals, name);
        Py_XINCREF(v);
    }
    else {
        v = PyObject_GetItem(locals, name);
        if (v == NULL) {
            if (!PyErr_ExceptionMatches(PyExc_KeyError))
                goto error;
            PyErr_Clear();
        }
    }
    /*global命名空间*/
    if (v == NULL) {
        v = PyDict_GetItem(f->f_globals, name);
        Py_XINCREF(v);
        /*builtin命名空间*/
        if (v == NULL) {
            if (PyDict_CheckExact(f->f_builtins)) {
                v = PyDict_GetItem(f->f_builtins, name);
                if (v == NULL) {
                    format_exc_check_arg(
                        PyExc_NameError,
                        NAME_ERROR_MSG, name);
                    goto error;
                }
                Py_INCREF(v);
            }
            else {
                v = PyObject_GetItem(f->f_builtins, name);
                if (v == NULL) {
                    if (PyErr_ExceptionMatches(PyExc_KeyError))
                        format_exc_check_arg(
                        PyExc_NameError,
                        NAME_ERROR_MSG, name);
                    goto error;
                }
            }
        }
    }
    PUSH(v);
    DISPATCH();
}
```

遵循LGB原则依次进行local、global和builtin命名空间的名字搜索，根据名字拿出对应的value。执行完以下两句字节码后：

```shell
4 LOAD_NAME                0 (a) 
6 LOAD_CONST               1 (10)
```

待比较的a和10都已经位于栈顶，此时可以执行下一句字节码：

```shell
8 COMPARE_OP               4 (>)
```

该字节码对应代码为：

```c
TARGET(COMPARE_OP) {
    PyObject *right = POP();
    PyObject *left = TOP();
    /*py2的实现是有对int类型（非大整数）做特殊处理的*/
    PyObject *res = cmp_outcome(oparg, left, right);
    Py_DECREF(left);
    Py_DECREF(right);
    SET_TOP(res);
    if (res == NULL)
        goto error;
    PREDICT(POP_JUMP_IF_FALSE);
    PREDICT(POP_JUMP_IF_TRUE);
    DISPATCH();
}
```

前面我们说到比较操作符的两个操作数都已经位于栈顶，通过定义的宏取出参数，这里实际只执行了一次`POP`，另一个参数是通过`TOP`得到的。

拿两个参数`right`和`left`进行比较，并用`SET_TOP`使用比较的结果`res`**挤掉**left在栈顶的位置。

##### 比较动作的执行者——cmp_outcome

接着虚拟机会调用`cmp_outcome`，其定义如下：

```c
static PyObject *
cmp_outcome(int op, PyObject *v, PyObject *w)
{
    int res = 0;
    switch (op) {
    case PyCmp_IS:
        res = (v == w);
        break;
    case PyCmp_IS_NOT:
        res = (v != w);
        break;
    case PyCmp_IN:
        res = PySequence_Contains(w, v);
        if (res < 0)
            return NULL;
        break;
    case PyCmp_NOT_IN:
        res = PySequence_Contains(w, v);
        if (res < 0)
            return NULL;
        res = !res;
        break;
    case PyCmp_EXC_MATCH:
        if (PyTuple_Check(w)) {
            Py_ssize_t i, length;
            length = PyTuple_Size(w);
            for (i = 0; i < length; i += 1) {
                PyObject *exc = PyTuple_GET_ITEM(w, i);
                if (!PyExceptionClass_Check(exc)) {
                    PyErr_SetString(PyExc_TypeError,
                                    CANNOT_CATCH_MSG);
                    return NULL;
                }
            }
        }
        else {
            if (!PyExceptionClass_Check(w)) {
                PyErr_SetString(PyExc_TypeError,
                                CANNOT_CATCH_MSG);
                return NULL;
            }
        }
        res = PyErr_GivenExceptionMatches(v, w);
        break;
    default:
        return PyObject_RichCompare(v, w, op);
    }
    v = res ? Py_True : Py_False;
    Py_INCREF(v);
    return v;
}
```

这里的比较包含了`is`、`is_not`等等，我们来重点看一看返回值：

```c
v = res ? Py_True : Py_False;
Py_INCREF(v);
return v;
```

`Py_True`和`Py_True`的定义如下：

```c
/* Use these macros */
#define Py_False ((PyObject *) &_Py_FalseStruct)
#define Py_True ((PyObject *) &_Py_TrueStruct)
```

事实上Python的布尔值是用long的0 1实现的：

```c
/* The objects representing bool values False and True */

struct _longobject _Py_FalseStruct = {
    PyVarObject_HEAD_INIT(&PyBool_Type, 0)
    { 0 }
};

struct _longobject _Py_TrueStruct = {
    PyVarObject_HEAD_INIT(&PyBool_Type, 1)
    { 1 }
};

```

并且这里的`Py_True`和`Py_True`都是全局的单例对象。

##### "可有可无"的PREDICT

注意这里的`PREDICT`宏：

> OpCode prediction macros
> Some opcodes tend to come in pairs thus making it possible to predict the second code when the first is run.  For example, COMPARE_OP is often followed by POP_JUMP_IF_FALSE or POP_JUMP_IF_TRUE.
>
> Verifying the prediction costs a single high-speed test of a register variable against a constant.  If the pairing was good, then the processor's own internal branch predication has a high likelihood of success, resulting in a nearly zero-overhead transition to the next opcode.  A successful prediction saves a trip through the eval-loop including its unpredictable switch-case branch.  
>
> Combined with the processor's internal branch prediction, a successful PREDICT has the effect of making the two opcodes run as if they were a single new opcode with the bodies combined.
>
> If collecting opcode statistics, your choices are to either keep the predictions turned-on and interpret the results as if some opcodes  had been combined or turn-off predictions so that the opcode frequency  counter updates for both opcodes. 
>
> Opcode prediction is disabled with threaded code, since the latter allows  the CPU to record separate branch prediction information for each opcode.

有些字节码一般是成对出现的，比如这里的`COMPARE_OP`和`POP_JUMP_IF_FALSE`。PREDICT定义如下：

```c
#if defined(DYNAMIC_EXECUTION_PROFILE) || USE_COMPUTED_GOTOS
#define PREDICT(op)             if (0) goto PRED_##op
#else
/*实际PREDICT为else分支的定义*/
#define PREDICT(op) \
    do{ \
        _Py_CODEUNIT word = *next_instr; \
        opcode = _Py_OPCODE(word); \
        if (opcode == op){ \
            oparg = _Py_OPARG(word); \
            next_instr++; \
            goto PRED_##op; \
        } \
    } while(0)
#endif
#define PREDICTED(op)           PRED_##op:
```

执行下列语句：

```c
PREDICT(POP_JUMP_IF_FALSE);
```

若下一条指令为`POP_JUMP_IF_FALSE`，则会在此直接增加"PC"指针（`next_instr`），并跳转到`PRED_##op`，省去外层的`eval-loop`：

>A successful prediction saves a trip through the eval-loop including its unpredictable switch-case branch.  

如果该操作码不是`POP_JUMP_IF_FALSE`，那么这里的`PREDICT`很明显是没有副作用的。

接着，`PRED_##op`其实就是宏拼接，在此展开其实就是`PRED_POP_JUMP_IF_FALSE`，这个label身在何处？自然是`POP_JUMP_IF_FALSE`的case处：

```c
PREDICTED(POP_JUMP_IF_FALSE);
TARGET(POP_JUMP_IF_FALSE) {
    PyObject *cond = POP();
    int err;
    if (cond == Py_True) {
        Py_DECREF(cond);
        FAST_DISPATCH();
    }
    if (cond == Py_False) {
        Py_DECREF(cond);
        JUMPTO(oparg);
        FAST_DISPATCH();
    }
    err = PyObject_IsTrue(cond);
    Py_DECREF(cond);
    if (err > 0)
        ;
    else if (err == 0)
        JUMPTO(oparg);
    else
        goto error;
    DISPATCH();
}
```

##### POP_JUMP_IF_FALSE字节码解析

上面的`PREDICTED(POP_JUMP_IF_FALSE)`展开后其实就是：

```c
PRED_POP_JUMP_IF_FALSE:
```

之前我们在`cmp_outcome`函数中将比较的结果放在了栈顶，这里pop出该结果`cond`，并进行逻辑的分发。当结果为true时，减少`cond`的引用计数并进行`FAST_DISPATCH`；当结果为false时，还会多做一个`JUMPTO(oparg)`的操作：

```c
#define JUMPTO(x)       (next_instr = first_instr + (x) / sizeof(_Py_CODEUNIT))
```

这里的 `oparg`其实就是字节码`10 POP_JUMP_IF_FALSE       22`中的22，这也正是else分支起始部分的额字节码**相对**`first_instr`的**偏移**地址。这里比较有趣的一点是对比较结果的判断并不是：

```c
if(cond == Py_True){
    
}else{
    
}
```

在`cond`既不是`Py_True`也不是`Py_False`时，会执行一个`PyObject_IsTrue`的动作：

```c
/* Test a value used as condition, e.g., in a for or if statement.
   Return -1 if an error occurred */

int
PyObject_IsTrue(PyObject *v)
{
    Py_ssize_t res;
    if (v == Py_True)
        return 1;
    if (v == Py_False)
        return 0;
    if (v == Py_None)
        return 0;
    else if (v->ob_type->tp_as_number != NULL &&
             v->ob_type->tp_as_number->nb_bool != NULL)
        res = (*v->ob_type->tp_as_number->nb_bool)(v);
    else if (v->ob_type->tp_as_mapping != NULL &&
             v->ob_type->tp_as_mapping->mp_length != NULL)
        res = (*v->ob_type->tp_as_mapping->mp_length)(v);
    else if (v->ob_type->tp_as_sequence != NULL &&
             v->ob_type->tp_as_sequence->sq_length != NULL)
        res = (*v->ob_type->tp_as_sequence->sq_length)(v);
    else
        return 1;
    /* if it is negative, it should be either -1 or -2 */
    return (res > 0) ? 1 : Py_SAFE_DOWNCAST(res, Py_ssize_t, int);
}
```

详细的逻辑不继续深挖，有兴趣可以自行了解（其实相当于cond不为"布尔值"时多做了一些逻辑判断）。

##### 其他

回到`PREDICT(op)`的定义，另一种情况是`if (0) goto PRED_##op`也就是一条空语句，这也正说明如果仅从功能实现的角度，PREDICT并不是必要的，只需要按部就班地解析指令即可。

```c
#if defined(DYNAMIC_EXECUTION_PROFILE) || USE_COMPUTED_GOTOS
#define PREDICT(op)             if (0) goto PRED_##op
#else
...
```

最后回到字节码`COMPARE_OP`，需要注意的是当下一条指令为`POP_JUMP_IF_FALSE`时，控制流是不会到后面的两条指令的：

```c
PREDICT(POP_JUMP_IF_FALSE);
PREDICT(POP_JUMP_IF_TRUE);
DISPATCH();
```

当比较结果为真时，控制流会顺序执行字节码：

```c
3           4 LOAD_NAME                0 (a) 
              6 LOAD_CONST               1 (10)
              8 COMPARE_OP               4 (>)
             10 POP_JUMP_IF_FALSE       22

  4          12 LOAD_NAME                1 (print)
             14 LOAD_CONST               2 ('a > 10')
             16 CALL_FUNCTION            1
             18 POP_TOP
             20 JUMP_FORWARD             8 (to 30)

  6     >>   22 LOAD_NAME                1 (print)
             24 LOAD_CONST               3 ('xixixi')
             26 CALL_FUNCTION            1
             28 POP_TOP
        >>   30 LOAD_CONST               4 (None)
             32 RETURN_VALUE
```

注意字节码`20 JUMP_FORWARD             8 (to 30)`，这是自然，毕竟我们只会执行一个分支的代码，因此我们执行完当前分支后会跳过else分支。