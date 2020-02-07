#### While循环

继for（范围）循环，我们再来看看while循环，python代码如下：

```python
i = 0
while i < 3:
    print(i)
    i += 1
```

得到字节码如下：

```shell
  6           0 LOAD_CONST               0 (0)
              2 STORE_NAME               0 (i)

  7           4 SETUP_LOOP              28 (to 34)
        >>    6 LOAD_NAME                0 (i)
              8 LOAD_CONST               1 (3)
             10 COMPARE_OP               0 (<)
             12 POP_JUMP_IF_FALSE       32

  8          14 LOAD_NAME                1 (print)
             16 LOAD_NAME                0 (i)
             18 CALL_FUNCTION            1
             20 POP_TOP

  9          22 LOAD_NAME                0 (i)
             24 LOAD_CONST               2 (1)
             26 INPLACE_ADD
             28 STORE_NAME               0 (i)
             30 JUMP_ABSOLUTE            6
        >>   32 POP_BLOCK
        >>   34 LOAD_CONST               3 (None)
             36 RETURN_VALUE

```

根据字节码来看while其实就是把if和[for循环](./py3-vm-loop.md)的逻辑放在了一起，在for中判断循环终止的逻辑事实上是在虚拟机实现一层（`next != NULL`不成立时循环结束）做的，而while的循环终止条件在这里是`i < 3`，这个逻辑自然就会很明显地出现在字节码中。

大部分字节码的逻辑其实在前面已经走过了，来看一看不认识的`INPLACE_ADD`，顾名思义，这应该是`+=`操作符产生的字节码，加法操作产生的结果不会新建对象，其对应代码如下：

```c
TARGET(INPLACE_ADD) {
    PyObject *right = POP();
    PyObject *left = TOP();
    PyObject *sum;
    if (PyUnicode_CheckExact(left) && PyUnicode_CheckExact(right)) {
        sum = unicode_concatenate(left, right, f, next_instr);
        /* unicode_concatenate consumed the ref to left */
    }
    else {
        sum = PyNumber_InPlaceAdd(left, right);
        Py_DECREF(left);
    }
    Py_DECREF(right);
    SET_TOP(sum);
    if (sum == NULL)
        goto error;
    DISPATCH();
}
```

这里是整数加法，所以会进入else分支，看看`PyNumber_InPlaceAdd`：

```c
PyObject *
PyNumber_InPlaceAdd(PyObject *v, PyObject *w)
{
    PyObject *result = binary_iop1(v, w, NB_SLOT(nb_inplace_add),
                                   NB_SLOT(nb_add));
    if (result == Py_NotImplemented) {
        PySequenceMethods *m = v->ob_type->tp_as_sequence;
        Py_DECREF(result);
        if (m != NULL) {
            binaryfunc f = NULL;
            f = m->sq_inplace_concat;
            if (f == NULL)
                f = m->sq_concat;
            if (f != NULL)
                return (*f)(v, w);
        }
        result = binop_type_error(v, w, "+=");
    }
    return result;
}
```

`PyNumber_InPlaceAdd`实际上调用了`binary_iop1`：

```c
static PyObject *
binary_iop1(PyObject *v, PyObject *w, const int iop_slot, const int op_slot)
{
    PyNumberMethods *mv = v->ob_type->tp_as_number;
    if (mv != NULL) {
        binaryfunc slot = NB_BINOP(mv, iop_slot);
        if (slot) {
            PyObject *x = (slot)(v, w);
            if (x != Py_NotImplemented) {
                return x;
            }
            Py_DECREF(x);
        }
    }
    return binary_op1(v, w, op_slot);
}
```

如果in-place操作下左操作数和加法结果是同一个object，那么为何我们还要再执行一个`STORE_NAME`呢？其实并不是所有的操作数都有in-place的操作，当无in-place操作就会使用non in-place的操作来代替执行：

```c
/* Binary in-place operators */

/* The in-place operators are defined to fall back to the 'normal',
   non in-place operations, if the in-place methods are not in place.

   - If the left hand object has the appropriate struct members, and
     they are filled, call the appropriate function and return the
     result.  No coercion is done on the arguments; the left-hand object
     is the one the operation is performed on, and it's up to the
     function to deal with the right-hand object.

   - Otherwise, in-place modification is not supported. Handle it exactly as
     a non in-place operation of the same kind.

   */

static PyObject *
binary_iop1(PyObject *v, PyObject *w, const int iop_slot, const int op_slot)
{
    ...
}
```

当把内层的`+=`操作符替换正常加法：

```c
i = 0
while i < 3:
    print(i)
    i = i + 1
```

字节码如下：

```shell
  6           0 LOAD_CONST               0 (0)
              2 STORE_NAME               0 (i)

  7           4 SETUP_LOOP              28 (to 34)
        >>    6 LOAD_NAME                0 (i)
              8 LOAD_CONST               1 (3)
             10 COMPARE_OP               0 (<)
             12 POP_JUMP_IF_FALSE       32

  8          14 LOAD_NAME                1 (print)
             16 LOAD_NAME                0 (i)
             18 CALL_FUNCTION            1
             20 POP_TOP

  9          22 LOAD_NAME                0 (i)
             24 LOAD_CONST               2 (1)
             26 BINARY_ADD
             28 STORE_NAME               0 (i)
             30 JUMP_ABSOLUTE            6
        >>   32 POP_BLOCK
        >>   34 LOAD_CONST               3 (None)
             36 RETURN_VALUE
```

`INPLACE_ADD`就替换为`BINARY_ADD`。