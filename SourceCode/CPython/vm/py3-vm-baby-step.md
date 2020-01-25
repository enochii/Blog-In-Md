### Python虚拟机框架

PyCodeObject：Python源代码经过编译之后，所有的字节码指令以及程序的其他**静态信息**都存放在PyCodeObject对象中。

PyFrameObject：执行环境，可以理解为一个栈帧。Python可以在这个执行环境中获取变量的值，也可以根据字节码指令的指示修改执行环境中某个变量的值，以影响后续的字节码指令。调用函数会在当前的执行环境之外重新创建一个新的执行环境。

```c
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

一个PyFrameObject有三个命名空间，命名空间简单来说可以认为就是一个定义约束的字典。

>PyObject * f_builtins;       / * builtin symbol table (PyDictObject) */
>PyObject * f_globals;        / *global symbol table (PyDictObject) */
>PyObject * f_locals;         / * local symbol table (any mapping) */

#### 第一张骨牌——PyEval_EvalFramEx

```c
/* 虚拟机执行字节码指令的基本架构 */
PyObject *
PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{
    PyThreadState *tstate = PyThreadState_GET();
    //PyInterpreterState
    return tstate->interp->eval_frame(f, throwflag);
}
```

两种状态，`PyInterpreterState`（表示一个进程）和`PyThreadState`（表示一个线程），其中IS中有类型为`eval_frame`的成员，顺藤摸瓜（好久）找到：

```c
/* 初始化一个IS!!! */
PyInterpreterState *
PyInterpreterState_New(void)
{
    ...
    interp->eval_frame = _PyEval_EvalFrameDefault;
    ...

    return interp;
}
```

由于`_PyEval_EvalFrameDefault`(`ceval.c`)实在是又臭又长，所以暂且来看看其内的一些注释：

```c
/* Computed GOTOs, or
       the-optimization-commonly-but-improperly-known-as-"threaded code"
   using gcc's labels-as-values extension (http://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html).

    字节码的eval采用循环内的switch
   The traditional bytecode evaluation loop uses a "switch" statement, which decent compilers will optimize as a single indirect branch instruction combined with a lookup table of jump addresses. However, since the indirect jump instruction is shared by all opcodes, the CPU will have a hard time making the right prediction for where to jump next (actually, it will be always wrong except in the uncommon case of a sequence of several identical opcodes).

   "Threaded code" in contrast, uses an explicit jump table and an explicit indirect jump instruction at the end of each opcode. Since the jump instruction is at a different address for each opcode, the CPU will make a separate prediction for each of these instructions, which is equivalent to predicting the second opcode of each opcode pair. These predictions have a much better chance to turn out valid, especially in small bytecode loops.

   A mispredicted branch on a modern CPU flushes the whole pipeline and can cost several CPU cycles (depending on the pipeline depth), and potentially many more instructions (depending on the pipeline width).
   A correctly predicted branch, however, is nearly free.

   At the time of this writing, the "threaded code" version is up to 15-20% faster than the normal "switch" version, depending on the compiler and the CPU architecture.

   We disable the optimization if DYNAMIC_EXECUTION_PROFILE is defined, because it would render the measurements invalid.


   NOTE: care must be taken that the compiler doesn't try to "optimize" the indirect jumps by sharing them between all opcodes. Such optimizations can be disabled on gcc by using the -fno-gcse flag (or possibly -fno-crossjumping).
*/
```

我觉得到这里就差不多了

>The traditional bytecode evaluation loop uses a "switch" statement

该函数内还有一个`why`变量，可以理解为异常🐎：

>enum why_code why; */\* Reason for block stack unwind \*/*

```c
/* Status code for main loop (reason for stack unwind) */
enum why_code {
        WHY_NOT =       0x0001, /* No error */
        WHY_EXCEPTION = 0x0002, /* Exception occurred */
        WHY_RETURN =    0x0008, /* 'return' statement */
        WHY_BREAK =     0x0010, /* 'break' statement */
        WHY_CONTINUE =  0x0020, /* 'continue' statement */
        WHY_YIELD =     0x0040, /* 'yield' operator */
        WHY_SILENCED =  0x0080  /* Exception silenced by 'with' */
};
```

#### 字节码解析工具

通过python自带的dis工具我们可以将python代码翻译为对应的字节码：

```python
import argparse
import dis


def file_to_bc(filename: str):
    s = open(file=filename).read()
    co = compile(s, filename, 'exec')
    dis.dis(co)

if __name__ == '__main__':
    parser = argparse.ArgumentParser(usage="it's usage tip.", description="help info.")
    parser.add_argument("--file")
 
    args = parser.parse_args()
    file_to_bc(args.file)
```

将其保存为`to_bc.py`，在命令行执行下列语句即可获得sample.py的字节码：

```shell
python_d.exe to_bc.py --file test/sample.py
```

