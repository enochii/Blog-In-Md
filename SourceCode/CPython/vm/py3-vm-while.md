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

