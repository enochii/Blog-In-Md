## Python中的多线程机制

#### GIL

在开始“深入”一些细节之前，先来探索下 GIL ，也就是 Global Interpreter Lock 。

> https://docs.python.org/3.7/c-api/init.html?highlight=gil#thread-state-and-the-global-interpreter-lock 
>
>  The Python interpreter is not fully thread-safe. In order to support multi-threaded Python programs, there’s a global lock, called the [global interpreter lock](https://docs.python.org/3.7/glossary.html#term-global-interpreter-lock) or [GIL](https://docs.python.org/3.7/glossary.html#term-gil), **that must be held by the current thread before it can safely access Python objects**. Without the lock, even the simplest operations could cause problems in a multi-threaded program: for example, when two threads simultaneously increment the reference count of the same object, the reference count could end up being incremented only once instead of twice. 

简单来说，一个 thread 在对一个 object 进行修改比如增减引用计数时需要 hold GIL。

这里也有个[有趣的问题]( https://stackoverflow.com/questions/26873512/why-does-python-provide-locking-mechanisms-if-its-subject-to-a-gil )，就是说为什么有了 GIL ，但 Python 还是会出现 data race 的现象。正如[轮子哥所说]( Python有GIL为什么还需要线程同步？ - vczh的回答 - 知乎 https://www.zhihu.com/question/23030421/answer/23475843 )：

> GIL也只是相当于古时候单核CPU通过不断的分配时间片来模拟多线程的方法而已，为什么那个时候写多线程也要用锁？ 

GIL 只能保证一条字节码的执行是原子的，而一些操作比如自增需要好几条字节码，故而会出现 data race 的现象。