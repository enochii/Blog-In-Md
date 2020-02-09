## Python 虚拟机中的类机制

这一部分一开始看得比较迷，基本是照着《Python 源码剖析》在整理自己的思绪。类机制中复杂的一块（我个人认为）在于继承以及继承带来的一些问题（比如操作重载之类），这部分如果靠自己头铁去看估计会被恶心死，比如就算现在大概有了一点思绪还是很疑惑为何*类的操作查找*会涉及一套如此复杂的机制，函数指针被包在 slot 中，slot 又被 descriptor 包了一层...

#### Python 中的对象模型

Python 2.2 之前存在的一个问题是，内置类型比如 int，dict，与自定义的 class 并不是完全等同的。**内置类型无法被继承。**

在面向对象的理论中，有两个核心的概念：类和对象。但如我们之前分析的源码中所示，我们的类型定义比如 PyList_Type 其实也是一种对象（object），list对象（list 的实例）则会有一个 type 指针指向这个（单例）类型对象。

在Python 2.2 之前，Python 存在三中对象：

- type 对象：内置类型
- class 对象：自定义类型
- instance 对象：由 class 创建的实例

在 2.2 之后，type 和 class 已经 统一，所以就剩下了 class 对象和 instance 对象。

> 在 class 对象和 class 对象之间，class 对象与instance 对象之间，存在着多种联系。我们将这些对象和他们之间的联系称为**类型系统或对象模型**。

##### 对象间的关系

- is kind of：子类与基类
- is instance of：类和实例

查看一个对象的 is-instance-of / is-kind-of关系：

- `.__class__`
- type()
- `.__bases__`
- issubclass
- isinstanceof

Python 2.2 之前存在的一个问题是，没有在 type 中寻找某个属性的机制，比如 `int.__add__`。2.2 之后每个类型对象（**PyTypeObject**的实例）中都会有 tp_dict，可用于查找 `__add__`对应的操作（函数指针），但这个机制并没有那么简单。

> callable，可调用性，只要一个对象对应的 class 实现了`__call__`操作（在PyTypeObject 中体现为 tp_call），那么这个对象就是一个可调用对象。

##### PyType_Ready

在 **PyType_Ready** 中，对类型信息做了一些处理。包括初始化基类：

```c

    /* Initialize tp_base (defaults to BaseObject unless that's us) */
    base = type->tp_base;
    if (base == NULL && type != &PyBaseObject_Type) {
        base = type->tp_base = &PyBaseObject_Type;
        Py_INCREF(base);
    }

    /* Now the only way base can still be NULL is if type is
     * &PyBaseObject_Type.
     * 只有 PyBaseObject_Type 的 base 才可以是 NULL
     */

    /* Initialize the base class
     * 是否初始化的标志是 base->tp_dict 是否为 NULL
     */
    if (base != NULL && base->tp_dict == NULL) {
        /* 用父类做参数，递归调用 */
        if (PyType_Ready(base) < 0)
            goto error;
    }
```

Python 支持多重继承，所以有一个基类列表 bases：

```c
	/* Initialize tp_bases */
    bases = type->tp_bases;
    if (bases == NULL) {
        if (base == NULL)
            bases = PyTuple_New(0);
        else
            bases = PyTuple_Pack(1, base);
        if (bases == NULL)
            goto error;
        type->tp_bases = bases;
    }
```

初始化 tp_dict：

```c
	/* Initialize tp_dict */
    dict = type->tp_dict;
    if (dict == NULL) {
        dict = PyDict_New();
        if (dict == NULL)
            goto error;
        type->tp_dict = dict;
    }
```

填充 tp_dict，在这个过程中，把类似("`__add__`", &nb_add)的元组放在 tp_dict 中

```c
 	/* Add type-specific descriptors to tp_dict */
    if (add_operators(type) < 0)
        goto error;
    if (type->tp_methods != NULL) {
        if (add_methods(type, type->tp_methods) < 0)
            goto error;
    }
    if (type->tp_members != NULL) {
        if (add_members(type, type->tp_members) < 0)
            goto error;
    }
    if (type->tp_getset != NULL) {
        if (add_getset(type, type->tp_getset) < 0)
            goto error;
    }
```

#### 填充 tp_dict

##### slot 与操作排序

```c
/*
Table mapping __foo__ names to tp_foo offsets and slot_tp_foo wrapper functions.

The table is ordered by offsets relative to the 'PyHeapTypeObject' structure,
which incorporates the additional structures used for numbers, sequences and
mappings.  Note that multiple names may map to the same slot (e.g. __eq__,
__ne__ etc. all map to tp_richcompare) and one name may map to multiple slots
(e.g. __str__ affects tp_str as well as tp_repr). The table is terminated with
an all-zero entry.  (This table is further initialized in init_slotdefs().)
*/

typedef struct wrapperbase slotdef;

struct wrapperbase {
    const char *name;
    int offset;
    void *function;
    wrapperfunc wrapper;
    const char *doc;
    int flags;
    PyObject *name_strobj;
};
```

在 Python 内部，slot 可以视为表示 PyTypeObject 定义的操作，一个操作对应一个 slot。slot 相关的宏定义有许多：

```c
#undef TPSLOT
#undef FLSLOT
#undef AMSLOT
#undef ETSLOT
#undef SQSLOT
#undef MPSLOT
#undef NBSLOT
#undef UNSLOT
#undef IBSLOT
#undef BINSLOT
#undef RBINSLOT

#define TPSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC) \
    {NAME, offsetof(PyTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC)}
#define FLSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC, FLAGS) \
    {NAME, offsetof(PyTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC), FLAGS}
#define ETSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC) \
    {NAME, offsetof(PyHeapTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC)}
...
```

TPSLOT 计算的是对应的函数指针（结构成员）在 PyTypeObject 中的偏移，ETSLOT 计算的则是该函数指针在 PyHeapTypeObject 中的偏移。

现在来理一下思绪，真正的操作（指向函数的指针）nb_add 是放在 PyNumberMethods 中，PyTypeObject 的实例中只是有指向 PyNumberMethods 结构体的指针，那么根据这些偏移信息最终要怎么拿到实际的函数指针？ 

先来看看 PyHeapTypeObject 的类型定义：

```c
/* The *real* layout of a type object when allocated on the heap */
typedef struct _heaptypeobject {
    /* Note: there's a dependency on the order of these members
       in slotptr() in typeobject.c . */
    PyTypeObject ht_type;
    PyAsyncMethods as_async;
    PyNumberMethods as_number;
    PyMappingMethods as_mapping;
    PySequenceMethods as_sequence; /* as_sequence comes after as_mapping,
                                      so that the mapping wins when both
                                      the mapping and the sequence define
                                      a given operator (e.g. __getitem__).
                                      see add_operators() in typeobject.c . */
    PyBufferProcs as_buffer;
    PyObject *ht_name, *ht_slots, *ht_qualname;
    struct _dictkeysobject *ht_cached_keys;
    /* here are optional user slots, followed by the members. */
} PyHeapTypeObject;
```

slotdef 前有这么一句注释，PyHeapTypeObject 只是为了确定 slotdefs table 的相对位置（具体机制后面会详细说明）：

```c
The table is ordered by offsets relative to the 'PyHeapTypeObject' structure
```

注意 slotdefs table，这里的 MPSLOT 和 SQSLOT 都是对 ETSLOT 进行了包装：

```c
static slotdef slotdefs[] = {
    ...
    MPSLOT("__getitem__", mp_subscript, slot_mp_subscript,
           wrap_binaryfunc,
           "__getitem__($self, key, /)\n--\n\nReturn self[key]."),
    ...
    SQSLOT("__getitem__", sq_item, slot_sq_item, wrap_sq_item,
           "__getitem__($self, key, /)\n--\n\nReturn self[key]."),
    ...    
}
```

注意到 slotdefs 中存在一个名字（比如 `__add__`）对应多个 slot 的情况（也就是存在一个名字下多个 slot 的相互竞争），而事实上就是来对应其优先级从而使得 python 得知应该调用哪个函数。

可以看到，在 init_slotdefs 之前被排好序了，在 init_slotdefs 中使用断言保证其 order（优先级）的正确性。

```c
static int slotdefs_initialized = 0;
/* Initialize the slotdefs table by adding interned string objects for the
   names. */
static void
init_slotdefs(void)
{
    slotdef *p;

    if (slotdefs_initialized)
        return;
    for (p = slotdefs; p->name; p++) {
        /* Slots must be ordered by their offset in the PyHeapTypeObject. */
        //这里
        assert(!p[1].name || p->offset <= p[1].offset);
        p->name_strobj = PyUnicode_InternFromString(p->name);
        if (!p->name_strobj || !PyUnicode_CHECK_INTERNED(p->name_strobj))
            Py_FatalError("Out of memory interning slotdef names");
    }
    slotdefs_initialized = 1;
}
```

##### 从 slot 到 descriptor

tp_dict 的键值应该都是 object，而 slot 并不是一个 object，所以最终放在 tp_dict 中的代表操作的东西其实是一个 descriptor，descriptor 是有多种类型的：

```c
typedef struct {
    PyObject_HEAD
    PyTypeObject *d_type;
    PyObject *d_name;
    PyObject *d_qualname;
} PyDescrObject;

#define PyDescr_COMMON PyDescrObject d_common

#define PyDescr_TYPE(x) (((PyDescrObject *)(x))->d_type)
#define PyDescr_NAME(x) (((PyDescrObject *)(x))->d_name)

typedef struct {
    PyDescr_COMMON;
    PyMethodDef *d_method;
} PyMethodDescrObject;

typedef struct {
    PyDescr_COMMON;
    struct PyMemberDef *d_member;
} PyMemberDescrObject;

typedef struct {
    PyDescr_COMMON;
    PyGetSetDef *d_getset;
} PyGetSetDescrObject;

typedef struct {
    PyDescr_COMMON;
    struct wrapperbase *d_base;
    void *d_wrapped; /* This can be any function pointer */
} PyWrapperDescrObject;
```

而对 slot 进行包装的正是 PyWrapperDescrObject。

##### 建立联系

前面提到 slotdefs 中的 slot 已经按照 PyHeapTypeObject 的意思进行了优先级排序，于是我们就可以在 type object 的 tp_dict 中建立操作名到 descriptor 的联系，这部分逻辑在 add_operators 中完成：

```c
static int
add_operators(PyTypeObject *type)
{
    PyObject *dict = type->tp_dict;
    slotdef *p;
    PyObject *descr;
    void **ptr;
	/*确保排序正确*/
    init_slotdefs();
    for (p = slotdefs; p->name; p++) {
        if (p->wrapper == NULL)
            continue;
        /*拿到函数指针*/
        ptr = slotptr(type, p->offset);
        if (!ptr || !*ptr)
            continue;
        /* 如果优先级更高的 descriptor已经放在了 tp_dict 中，就直接跳过 */
        if (PyDict_GetItem(dict, p->name_strobj))
            continue;
        if (*ptr == (void *)PyObject_HashNotImplemented) {
            /* Classes may prevent the inheritance of the tp_hash
               slot by storing PyObject_HashNotImplemented in it. Make it
               visible as a None value for the __hash__ attribute. */
            if (PyDict_SetItem(dict, p->name_strobj, Py_None) < 0)
                return -1;
        }
        else {
            /*建立descriptor*/
            descr = PyDescr_NewWrapper(type, p, *ptr);
            if (descr == NULL)
                return -1;
            /*放入tp_dict中*/
            if (PyDict_SetItem(dict, p->name_strobj, descr) < 0) {
                Py_DECREF(descr);
                return -1;
            }
            Py_DECREF(descr);
        }
    }
    if (type->tp_new != NULL) {
        if (add_tp_new_wrapper(type) < 0)
            return -1;
    }
    return 0;
}
```

同时注意这里是按照 slotdefs 的顺序对 slot 进行遍历的，同时在新建 descriptor 并放入 tp_dict 时会检查对应的 name_strobj 是否已经绑定了一个 descriptor，如果 PyDict_GetItem 的结果不为空，证明有优先级更高的 descriptor 已经绑定了该名字，这样一来 tp_dict 便能正确拿到更高优先级的 descriptor。

```c
if (PyDict_GetItem(dict, p->name_strobj))
            continue;
```

有兴趣也可以看看 add_operator 前的注释，说了我们费尽口舌所要表达的机理：

```c
/* This function is called by PyType_Ready() to populate the type's
   dictionary with method descriptors for function slots.  For each
   function slot (like tp_repr) that's defined in the type, one or more
   corresponding descriptors are added in the type's tp_dict dictionary
   under the appropriate name (like __repr__).  Some function slots
   cause more than one descriptor to be added (for example, the nb_add
   slot adds both __add__ and __radd__ descriptors) and some function
   slots compete for the same descriptor (for example both sq_item and
   mp_subscript generate a __getitem__ descriptor).

   In the latter case, the first slotdef entry encountered wins.  Since
   slotdef entries are sorted by the offset of the slot in the
   PyHeapTypeObject, this gives us some control over disambiguating
   between competing slots: the members of PyHeapTypeObject are listed
   from most general to least general, so the most general slot is
   preferred.  In particular, because as_mapping comes before as_sequence,
   for a type that defines both mp_subscript and sq_item, mp_subscript
   wins.

   This only adds new descriptors and doesn't overwrite entries in
   tp_dict that were previously defined.  The descriptors contain a
   reference to the C function they must call, so that it's safe if they
   are copied into a subtype's __dict__ and the subtype has a different
   C function in its slot -- calling the method defined by the
   descriptor will call the C function that was used to create it,
   rather than the C function present in the slot when it is called.
   (This is important because a subtype may have a C function in the
   slot that calls the method from the dictionary, and we want to avoid
   infinite recursion here.) */
```

慢着，我们是如何拿到函数指针的？毕竟前面花了好大的弯子在讲述 操作相对于 type object 以及 heap object 的偏移。玄机隐藏在 slotptr 中：

```c
ptr = slotptr(type, p->offset);
```

```c
/* Given a type pointer and an offset gotten from a slotdef entry, return a
   pointer to the actual slot.  This is not quite the same as simply adding
   the offset to the type pointer, since it takes care to indirect through the
   proper indirection pointer (as_buffer, etc.); it returns NULL if the
   indirection pointer is NULL. */
static void **
slotptr(PyTypeObject *type, int ioffset)
{
    char *ptr;
    long offset = ioffset;

    /* Note: this depends on the order of the members of PyHeapTypeObject! */
    assert(offset >= 0);
    assert((size_t)offset < offsetof(PyHeapTypeObject, as_buffer));
    if ((size_t)offset >= offsetof(PyHeapTypeObject, as_sequence)) {
        ptr = (char *)type->tp_as_sequence;
        offset -= offsetof(PyHeapTypeObject, as_sequence);
    }
    else if ((size_t)offset >= offsetof(PyHeapTypeObject, as_mapping)) {
        ptr = (char *)type->tp_as_mapping;
        offset -= offsetof(PyHeapTypeObject, as_mapping);
    }
    else if ((size_t)offset >= offsetof(PyHeapTypeObject, as_number)) {
        ptr = (char *)type->tp_as_number;
        offset -= offsetof(PyHeapTypeObject, as_number);
    }
    else if ((size_t)offset >= offsetof(PyHeapTypeObject, as_async)) {
        ptr = (char *)type->tp_as_async;
        offset -= offsetof(PyHeapTypeObject, as_async);
    }
    else {
        ptr = (char *)type;
    }
    if (ptr != NULL)
        ptr += offset;
    return (void **)ptr;
}
```

我们在 slotdef 中放的 offset 是函数指针对应的结构成员相对于 PyHeapTypeObject 的偏移，这里由于我们是要根据这个 offset 去做 >= 判断该拿哪个 slot，所以应该是从后往前（优先级从低到高）判断。

当知道我们要的是 PyHeapTypeObject 的哪个成员比如 as_mapping，执行`offset -= offsetof(PyHeapTypeObject, as_mapping)`后这个 offset 就是该操作相对于 as_mapping 的偏移，`ptr += offset`后就指向了type(类型对象)对应的结构成员，这样就拿到了我们要的函数指针（比如 `__add__`）。

##### 操作的覆盖（重定义）待补充

#### 类继承相关

##### 确定 MRO

MRO，即 Method Resolve Order，也就是一个 class 对象的属性解析顺序。因为 python 支持多重继承，就必须设置按照何种顺序解析属性。

在 PyType_Ready 中，通过 mro_internal 完成相关工作：

```c
	/* Calculate method resolution order */
    if (mro_internal(type, NULL) < 0)
        goto error;
```

暂时可以简单认为 mro 是一个列表。

##### 继承基类操作

在 PyType_Ready 中，会根据 mro 列表初始化基类：

```c
	/* Initialize tp_dict properly */
    bases = type->tp_mro;
    assert(bases != NULL);
    assert(PyTuple_Check(bases));
    n = PyTuple_GET_SIZE(bases);
    for (i = 1; i < n; i++) {
        PyObject *b = PyTuple_GET_ITEM(bases, i);
        if (PyType_Check(b))
            inherit_slots(type, (PyTypeObject *)b);
    }
```

从 1 开始也即是 mro 列表的第二项开始遍历是因为第一项其实是该类型自身，对于每个基类是通过一个`inherit_slots(type, (PyTypeObject *)b)`调用去进行继承操作的。

在 inherit_slots 中：

```c
	#define COPYSLOT(SLOT) \
    if (!type->SLOT && SLOTDEFINED(SLOT)) type->SLOT = base->SLOT

    #define COPYASYNC(SLOT) COPYSLOT(tp_as_async->SLOT)
    #define COPYNUM(SLOT) COPYSLOT(tp_as_number->SLOT)
	...
	/* This won't inherit indirect slots (from tp_as_number etc.)
       if type doesn't provide the space. */

    if (type->tp_as_number != NULL && base->tp_as_number != NULL) {
        basebase = base->tp_base;
        if (basebase->tp_as_number == NULL)
            basebase = NULL;
        COPYNUM(nb_add);
        COPYNUM(nb_subtract);
        ...
    }
    ...
```

##### 填充基类中的子类列表

```c
	/* Link into each base class's list of subclasses */
    bases = type->tp_bases;
    n = PyTuple_GET_SIZE(bases);
    for (i = 0; i < n; i++) {
        PyObject *b = PyTuple_GET_ITEM(bases, i);
        if (PyType_Check(b) &&
            add_subclass((PyTypeObject *)b, type) < 0)
            goto error;
    }
```

> 在 PyType_Ready 中 bases 局部变量有时是 tp->tp_bases，有时候是 mro 列表。
>
> 后面留点心看看 tp_bases 和 tp_mro 的关系？

PyType_Ready 中对类型对象做的初始化工作大致包括：

- 设置 type 信息、基类及基类列表
- 填充 tp_dict
- 确定 mro 列表
- 基于 mro 列表从基类继承操作
- 设置基类的子类列表

