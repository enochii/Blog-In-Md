## 异常

### 虚拟机对异常的基本支持

这部分来探索下虚拟机中对异常的支持，代码如下：

```python
1 / 0
```

抛出异常：

```shell
$  ./python_d.exe test/exception.py
   >Traceback (most recent call last):
      File "test/exception.py", line 1, in <module>
        1 / 0
    ZeroDivisionError: division by zero
```

字节码如下：

```c
  1           0 LOAD_CONST               0 (1)
              2 LOAD_CONST               1 (0)
              4 BINARY_TRUE_DIVIDE
              6 POP_TOP
              8 LOAD_CONST               2 (None)
             10 RETURN_VALUE
```

`BINARY_TRUE_DIVIDE`对应代码实现如下：

```c
TARGET(BINARY_TRUE_DIVIDE) {
    PyObject *divisor = POP();
    PyObject *dividend = TOP();
    PyObject *quotient = PyNumber_TrueDivide(dividend, divisor);
    Py_DECREF(dividend);
    Py_DECREF(divisor);
    SET_TOP(quotient);
    if (quotient == NULL)
        goto error;
    DISPATCH();
}
```

`PyNumber_TrueDivide`最终会调用`long_as_number`的`long_true_divide`，`long_as_number`即long类型实现的`PythonNumberMethod`。

```c
static PyNumberMethods long_as_number = {
    ...
    long_div,                   /* nb_floor_divide */
    long_true_divide,           /* nb_true_divide */
    0,                          /*nb_inplace_floor_divide */
    0,                          /* nb_inplace_true_divide */
    long_long,                  /* nb_index */
};
```

`long_true_divide`定义如下：

```c
static PyObject *
long_true_divide(PyObject *v, PyObject *w)
{
    ...
    if (b_size == 0) {
        /* 除0错误 */
        PyErr_SetString(PyExc_ZeroDivisionError,
                        "division by zero");
        goto error;
    }
    ...
}
```

其中`PyExc_ZeroDivisionError`为异常类型对象，`PyErr_SetString`最终会调用`PyErr_Restore`，在当前线程状态中保存`exception`、`value`和`traceback`对象。

```c
void
PyErr_SetString(PyObject *exception, const char *string)
{
    PyObject *value = PyUnicode_FromString(string);
    PyErr_SetObject(exception, value);
    Py_XDECREF(value);
}
```

```c
void
PyErr_SetObject(PyObject *exception, PyObject *value)
{
    ...
        // tb -> traceback
    PyErr_Restore(exception, value, tb);
}
```

```c
void
PyErr_Restore(PyObject *type, PyObject *value, PyObject *traceback)
{
    PyThreadState *tstate = PyThreadState_GET();
    PyObject *oldtype, *oldvalue, *oldtraceback;

    if (traceback != NULL && !PyTraceBack_Check(traceback)) 	{
        /* XXX Should never happen -- fatal error instead?*/
        /* Well, it could be None. */
        Py_DECREF(traceback);
        traceback = NULL;
    }

    /* Save these in locals to safeguard against recursive
       invocation through Py_XDECREF */
    oldtype = tstate->curexc_type;
    oldvalue = tstate->curexc_value;
    oldtraceback = tstate->curexc_traceback;

    tstate->curexc_type = type;
    tstate->curexc_value = value;
    tstate->curexc_traceback = traceback;

    Py_XDECREF(oldtype);
    Py_XDECREF(oldvalue);
    Py_XDECREF(oldtraceback);
}
```

接着`long_true_divide`会通过`goto error`语句跳转到错误处理：

```c
error:
    /* 异常处理（goto error） */
    assert(why == WHY_NOT);
    /* 设置 WHY_EXCEPTION 标志*/
    why = WHY_EXCEPTION;
	...

    /* pop frame */
exit_eval_frame:
    if (PyDTrace_FUNCTION_RETURN_ENABLED())
        dtrace_function_return(f);
    Py_LeaveRecursiveCall();
    f->f_executing = 0;
    /* 返回上一个栈帧 */
    tstate->frame = f->f_back;

    return _Py_CheckFunctionResult(NULL, retval, "PyEval_EvalFrameEx");
}/*end of eval_frame*/
```

如果在当前栈帧无法捕捉异常，那么会重新设置当前线程状态对象中活动栈帧(`tstate->frame = f->f_back`)，完成栈帧回退的动作。

### 异常语义控制结构

Python代码如下：

```python
try:
    raise Exception('Im a E')

except Exception as e:
    print(e)

```

得到字节码：

```shell
  			  # 创建一个block，链接到catch块
  3           0 SETUP_EXCEPT            12 (to 14)

  4           2 LOAD_NAME                0 (Exception)
              4 LOAD_CONST               0 ('Im a E')
              6 CALL_FUNCTION            1
              
              # 调用do_raise()，并会在tsate记录当前的 异常信息
              # 也即PyErr_Restore()
              # 这里还会跳转到 error标签，调用PyErr_Fetch
              # 最终回来到 字节码14
              8 RAISE_VARARGS            1
             10 POP_BLOCK
             12 JUMP_FORWARD            34 (to 48)
			
  5     >>   14 DUP_TOP
             16 LOAD_NAME                0 (Exception)
             18 COMPARE_OP              10 (exception match)
             20 POP_JUMP_IF_FALSE       46
             22 POP_TOP
             24 STORE_NAME               1 (e)
             26 POP_TOP
             28 SETUP_FINALLY            4 (to 34)
				
			 # 这里空白对应pass
			
  6          30 POP_BLOCK
             32 LOAD_CONST               1 (None)
             
             # 34 36感觉无用功
        >>   34 LOAD_CONST               1 (None)
             36 STORE_NAME               1 (e)
             
             # e离开了作用域
             38 DELETE_NAME              1 (e)
             
             # 这里的END_FINALLY会进入 POP()为NULL的分支
             # 因为 字节码32 压入了一个 None值
             40 END_FINALLY
             
             # 清除栈中的 异常信息 (tb type value)
             42 POP_EXCEPT
             44 JUMP_FORWARD             2 (to 48)
             # 如果不匹配，这里会再次调用PyErr_Restore()
             # 因为栈顶 是一个 exception
        >>   46 END_FINALLY
        >>   48 LOAD_CONST               1 (None)
             50 RETURN_VALUE

```

以下三句字节码首先创建了一个异常对象，并将其压栈：

```shell
  4           4 LOAD_NAME                0 (Exception)
              6 LOAD_CONST               0 ('Im a E')
              8 CALL_FUNCTION            1
```

先看这句字节码`RAISE_VARARGS		1`：

```c
TARGET(RAISE_VARARGS) {
            PyObject *cause = NULL, *exc = NULL;
            switch (oparg) {
            case 2:
                cause = POP(); /* cause */
                /* fall through */
            case 1:
                exc = POP(); /* exc */
                /* fall through */
            case 0:
                if (do_raise(exc, cause)) {
                    why = WHY_EXCEPTION;
                    goto fast_block_end;
                }
                break;
            default:
                PyErr_SetString(PyExc_SystemError,
                           "bad RAISE_VARARGS oparg");
                break;
            }
            goto error;
        }
```

`do_raise`如下，由于这里`oparg`为1，所以就是调用`do_raise(exp, NULL);`。

```c
/* Logic for the raise statement (too complicated for inlining).
   This *consumes* a reference count to each of its arguments. */
static int
do_raise(PyObject *exc, PyObject *cause)
{
    ...
    else if (PyExceptionInstance_Check(exc)) {
        value = exc;
        type = PyExceptionInstance_Class(exc);
        Py_INCREF(type);
    }
    ...
    
	// 这里会调用PyErr_Restore()在线程状态中记录异常信息
    PyErr_SetObject(type, value);
    /* PyErr_SetObject incref's its arguments */
    Py_DECREF(value);
    Py_DECREF(type);
    
    return 0;
    ...
}
```

注意这里调用了`PyErr_SetObject()`，其最终会调用`PyErr_Restore()`在线程状态中记录异常信息。另外，根据字节码（参数）判断最终该函数会返回0，最终会执行`goto error;` 在其中有一个栈帧展开的循环：

```c
/* Unwind stacks if a (pseudo) exception occurred */
        while (why != WHY_NOT && f->f_iblock > 0) {
            /* Peek at the current block. */
            PyTryBlock *b = &f->f_blockstack[f->f_iblock - 1];

            ...
            /* Now we have to pop the block. */
            f->f_iblock--;

            ...
            if (why == WHY_EXCEPTION && (b->b_type == SETUP_EXCEPT || b->b_type == SETUP_FINALLY)) {
                PyObject *exc, *val, *tb;
                int handler = b->b_handler;
                _PyErr_StackItem *exc_info = tstate->exc_info;
                /* Beware, this invalidates all b->b_* fields */
                PyFrame_BlockSetup(f, EXCEPT_HANDLER, -1, STACK_LEVEL());
                PUSH(exc_info->exc_traceback);
                PUSH(exc_info->exc_value);
                if (exc_info->exc_type != NULL) {
                    PUSH(exc_info->exc_type);
                }
                else {
                    Py_INCREF(Py_None);
                    PUSH(Py_None);
                }
                /*从线程状态中获取 异常信息*/
                PyErr_Fetch(&exc, &val, &tb);
                /* Make the raw exception data
                   available to the handler,
                   so a program can emulate the
                   Python main loop. */
                PyErr_NormalizeException(
                    &exc, &val, &tb);
                if (tb != NULL)
                    PyException_SetTraceback(val, tb);
                else
                    PyException_SetTraceback(val, Py_None);
                Py_INCREF(exc);
                exc_info->exc_type = exc;
                Py_INCREF(val);
                exc_info->exc_value = val;
                exc_info->exc_traceback = tb;
                if (tb == NULL)
                    tb = Py_None;
                Py_INCREF(tb);
                PUSH(tb);
                PUSH(val);
                PUSH(exc);
                why = WHY_NOT;
                /* 跳转到except的字节码 */
                // printf("handler: %d\n");
                JUMPTO(handler);
                break;
            }
```

这里会调用`PyErr_Fetch()`从线程状态中获取异常信息，相当于是`PyErr_Restore()`的逆动作，并将他们**压栈**，同时修改`why`变量的状态，并通过`JUMPTO(handler)`跳转到第一条字节码`SETUP_EXCEPT`设置的字节码地址(14)，也就是我们自定义的异常处理代码。

```shell
			 14 DUP_TOP
             16 LOAD_NAME                0 (Exception)
             18 COMPARE_OP              10 (exception match)
             20 POP_JUMP_IF_FALSE       46
             22 POP_TOP
             24 STORE_NAME               1 (e)
             26 POP_TOP
             28 SETUP_FINALLY            4 (to 34)
```

这里判断是否成功捕捉异常，如果成功，会通过`SETUP_FINALLY()`新起一个block，对应到python代码也就是异常处理逻辑 (pass)。当捕捉异常失败时，会跳转到46，否则会执行到44然后通过`JUMP_FORWORD`跳转到48，也就是说是否捕捉异常会决定字节码46是否被执行。这个字节码46事实上会调用`PyErr_Restore`，因为异常没有被正确捕捉，需要重新保存在线程状态中，以便后续的异常处理。

### 小结

虚拟机的异常机制大致就是，会在当前栈帧寻找handler，若无法捕捉异常则会执行栈帧的回退。异常信息(`tb` `value` `type`)有可能会在运行时栈也可能位于线程状态内。

Block有三种 

- LOOP
- FINALLY
- EXCEPT

事实上他们都是为了实现字节码的跳转，比如while中的`break`。



这里不明白的一点是生成的字节码中有两次`STORE_NAME`，并且有一次`DELETE_NAME`。