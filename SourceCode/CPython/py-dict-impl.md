## Python Dictionary Implementation

最近跟着[这个项目]( https://flaggo.github.io/python3-source-code-analysis/objects/dict-object/ )看cpython的源码，一开始是想照着陈儒老师的《Python源码剖析》看看2的实现，编译项目需要用到Visual Studio，貌似2.5支持的版本是08和05的... 我用19一直卡着，后面就直接换成3.7.0的，比较之顺利。由于开源项目对源码的剖析并不是很详细，所以基本是同时参考了两块，这里感觉可以写点东西，于是照着自己的理解来做下记录。

> 文章基于3.7.0的dict实现进行分析，可能会稍微提下2的一些东西。阅读此文大概要对Python的对象机制有一点点的了解。

#### 概要

Python的`dict`实现事实上就是做了一个hash table，STL的`map`则大不相同，其底层采用了红黑树，好坏限于个人知识在此按下不表，另外事实上STL也是给了一个哈希表的实现的（`unordered_map`）。

##### 冲突的解决

我们知道，对于哈希而言，冲突是不可避免的，常用的解决办法有开链法、线性探测法和平方探测法，Python实现上采用了平方探测法来解决冲突。这意味着不同的key值映射到同一个hash值上后会形成一条”链“，所以在删除时只能采用**懒（伪）删除**进行标记。

#### 容器相关数据结构（PyDictObject和PyDictKeysObject）

##### 字典的entry——PyDictKeyEntry 

PyDictKeyEntry代表字典中的一个entry，其定义如下

````c
typedef struct {
    /* Cached hash code of me_key. */
    Py_hash_t me_hash;
    PyObject *me_key;
    PyObject *me_value; /* This field is only meaningful for combined tables */
} PyDictKeyEntry;
```

`me_hash`用来缓存哈希值避免多次计算。

##### Key-Sharing Dictionary

Python的字典事实上是有两种类型的，分别是`split`和`combined`表，这里的分离指的是key和value值，可以看看[官方介绍]( https://www.python.org/dev/peps/pep-0412/ )，更具普适性的是combined类型的dict，后面也会主要（基本只）围绕其展开。

```shell
/*
The DictObject can be in one of two forms.

Either:
  A combined table:
    ma_values == NULL, dk_refcnt == 1.
    Values are stored in the me_value field of the PyDictKeysObject.
Or:
  A split table:
    ma_values != NULL, dk_refcnt >= 1
    Values are stored in the ma_values array.
    Only string (unicode) keys are allowed.
    All dicts sharing same key must have same insertion order.

There are four kinds of slots in the table (slot is index, and
DK_ENTRIES(keys)[index] if index >= 0):
*/
```

##### 字典的外壳——PyDictObject

字典PyDictObject的定义如下：

```c
/* The ma_values pointer is NULL for a combined table
 * or points to an array of PyObject* for a split table
 */
/*
 * 再出现split表之前是有一个small_table的
 * 当字典的大小比较小（<8）时，就不会去alloc一个动态的表
 * 出现split表和combine表概念之后，就不能这么做了
 */ 
typedef struct {
    PyObject_HEAD

    /* Number of items in the dictionary */
    Py_ssize_t ma_used;

    /* Dictionary version: globally unique, value change each time
       the dictionary is modified */
    uint64_t ma_version_tag;

    PyDictKeysObject *ma_keys;

    /* If ma_values is NULL, the table is "combined": keys and values
       are stored in ma_keys.

       If ma_values is not NULL, the table is splitted:
       keys are stored in ma_keys and values are stored in ma_values 
       
       如果是combined表，那么ma_values为NULL，且值都在ma_keys中
       */
    PyObject **ma_values;
} PyDictObject;

```

首先是一个定长对象`PyObject_HEAD`，也就是：

```c
//object.h
/* PyObject_HEAD defines the initial segment of every PyObject. */
#define PyObject_HEAD                   PyObject ob_base;
```

注释已经说得很清楚了，当其为`split`类型时，key和value分别在`ma_keys`和`ma_values`中；当为`combined`类型时，键值对都是在`ma_keys`中的，且`ma_values==NULL`，这个也可以作为判断字典类型的依据。

##### PyDictEntry的状态

entry有四个状态，分别是

1. Unused.  index == DKIX_EMPTY
   Does not hold an active (key, value) pair now and never did.  Unused can
   transition to Active upon key insertion.  This is each slot's initial state.
2. Active.  index >= 0, me_key != NULL and me_value != NULL
   Holds an active (key, value) pair.  Active can transition to Dummy or
   Pending upon key deletion (for combined and split tables respectively).
   This is the only case in which me_value != NULL.
3. Dummy.  index == DKIX_DUMMY  (combined only)
   Previously held an active (key, value) pair, but that was deleted and an
   active pair has not yet overwritten the slot.  Dummy can transition to
   Active upon key insertion.  Dummy slots cannot be made Unused again
   else the probe sequence in case of collision would have no way to know
   they were once active.
4. Pending. index >= 0, key != NULL, and value == NULL  (split only)
   Not yet inserted in split-table.

事实上具体到某个entry，它只有三种状态，因为Dummy和Pending状态都是属于具体类型的哈希表（分别是combined和split）。前面提到entry的删除是懒删除，所以在删除后entry就转成了Dummy态（combined类型dict）。

##### PyDictKeysObject的内存布局

因为我们主要是考察`combined`类型的dict，所以我们先把`ma_values`放在一边，来康康`ma_keys`，这才是内部哈希表的核心实现，其类型`PyDictKeysObject`定义如下：

```c
/* See dictobject.c for actual layout of DictKeysObject */
struct _dictkeysobject {
    Py_ssize_t dk_refcnt;

    /* Size of the hash table (dk_indices). It must be a power of 2. */
    Py_ssize_t dk_size;

    /* Function to lookup in the hash table (dk_indices):

       - lookdict(): general-purpose, and may return DKIX_ERROR if (and
         only if) a comparison raises an exception.

       - lookdict_unicode(): specialized to Unicode string keys, comparison of
         which can never raise an exception; that function can never return
         DKIX_ERROR.

       - lookdict_unicode_nodummy(): similar to lookdict_unicode() but further
         specialized for Unicode string keys that cannot be the <dummy> value.

       - lookdict_split(): Version of lookdict() for split tables. */
    dict_lookup_func dk_lookup;

    /* Number of usable entries in dk_entries. */
    Py_ssize_t dk_usable;

    /* Number of used entries in dk_entries. */
    Py_ssize_t dk_nentries;

    /* Actual hash table of dk_size entries. It holds indices in dk_entries,
       or DKIX_EMPTY(-1) or DKIX_DUMMY(-2).

       Indices must be: 0 <= indice < USABLE_FRACTION(dk_size).

       The size in bytes of an indice depends on dk_size:

       - 1 byte if dk_size <= 0xff (char*)
       - 2 bytes if dk_size <= 0xffff (int16_t*)
       - 4 bytes if dk_size <= 0xffffffff (int32_t*)
       - 8 bytes otherwise (int64_t*)

       Dynamically sized, SIZEOF_VOID_P is minimum. */
    char dk_indices[];  /* char is required to avoid strict aliasing. */

    /* "PyDictKeyEntry dk_entries[dk_usable];" array follows:
       see the DK_ENTRIES() macro */
};
```

主要看`char dk_indices[]`，这是C的`flexsible array`也就是柔性数组，其长度不定。遵循注释，我们来看看`DictKeysObject`的内存布局：

```c
//dictobject.c
layout:

+---------------+
| dk_refcnt     |
| dk_size       |
| dk_lookup     |
| dk_usable     |
| dk_nentries   |
+---------------+
| dk_indices    |
|               |
+---------------+
| dk_entries    |
|               |
+---------------+
```

可以看到，所谓的柔性数组在结构体后面不仅藏了一个`dk_indices`索引数组，`dk_entries`数组也紧随其后。大概哈希表在这里的逻辑是，一个值经过哈希函数F(x)后会拿到一个i，这个i是dk_indices数组的下标，通过`dk_entries[dk_indices[i]]`就可以拿到对应的entry，细节在后面会展开。

##### DK_ENTRIES——拿出entry数组

因为在`DictKeysObject`中`dk_indices`作为一个柔性数组事实上是包含了`dk_indices`和`dk_entries`的，如果要拿到`dk_entries`，需要做一下指针的偏移，这边也是提供了一个宏：

```c
#define DK_ENTRIES(dk) \
    ((PyDictKeyEntry*)(&((int8_t*)((dk)->dk_indices))[DK_SIZE(dk) * DK_IXSIZE(dk)]))
```

> 这里明明提供了`DK_IXSIZE`这种好用的东西来计算dk的索引数组的元素size，其他的一些地方好像没用...

#### 字典的操作

好了，知道了这大概是一个什么逻辑后，我们来看看字典的一些操作。

##### 创建

首先是创建一个(combined)字典：

```c
PyObject *
PyDict_New(void)
{
    PyDictKeysObject *keys = new_keys_object(PyDict_MINSIZE);
    if (keys == NULL)
        return NULL;
    //做一个combined类型的字典（ma_values为NULL）
    return new_dict(keys, NULL);
}
```

可以看到新建一个字典分为两步，先`new_keys_object（）`创建内部哈希表，然后`new_dict（）`创建一个字典。先康康`new_keys = new_keys_object(newsize)`：

```c
static PyDictKeysObject *new_keys_object(Py_ssize_t size)
{
    PyDictKeysObject *dk;
    Py_ssize_t es, usable;

    assert(size >= PyDict_MINSIZE);
    assert(IS_POWER_OF_2(size));

    /*
     * usable事实上是entry的数量
     * size是indices的数量
     * 这就是那个装载率的问题
     * 下面申请内存就是这样做的
    */
    usable = USABLE_FRACTION(size);
    if (size <= 0xff) {
        es = 1;
    }
    else if (size <= 0xffff) {
        es = 2;
    }
#if SIZEOF_VOID_P > 4
    else if (size <= 0xffffffff) {
        es = 4;
    }
#endif
    else {
        es = sizeof(Py_ssize_t);
    }

    if (size == PyDict_MINSIZE && numfreekeys > 0) {
        dk = keys_free_list[--numfreekeys];
    }
    else {
        /* 这里申请内存
         * 固定长度 + 柔性数组（indices + entries）
         * size是索引数组的长度
         * usable是entry数组的长度
         */
        dk = PyObject_MALLOC(sizeof(PyDictKeysObject)
                             + es * size
                             + sizeof(PyDictKeyEntry) * usable);
        if (dk == NULL) {
            PyErr_NoMemory();
            return NULL;
        }
    }
    DK_DEBUG_INCREF dk->dk_refcnt = 1;
    
    dk->dk_size = size;
    dk->dk_usable = usable;
    dk->dk_lookup = lookdict_unicode_nodummy;
    dk->dk_nentries = 0;
    //清空内存
    memset(&dk->dk_indices[0], 0xff, es * size);
    memset(DK_ENTRIES(dk), 0, sizeof(PyDictKeyEntry) * usable);
    return dk;
}
```

这里有一个**装载率**的问题，哈希中的装载率指的是表中 真实存在的entry数 / entry总数，实践中1/2或2/3是比较适宜的，在创建`PyDictKeysObject`时，`size`是哈希表的大小，`usable`是真实entry的数量。

```c
usable = USABLE_FRACTION(size);
```

`USABLE_FRACTION`宏定义如下：

```c
/* USABLE_FRACTION is the maximum dictionary load.
 * Increasing this ratio makes dictionaries more dense resulting in more
 * collisions.  Decreasing it improves sparseness at the expense of spreading
 * indices over more cache lines and at the cost of total memory consumed.
 *
 * USABLE_FRACTION must obey the following:
 *     (0 < USABLE_FRACTION(n) < n) for all n >= 2
 *
 * USABLE_FRACTION should be quick to calculate.
 * Fractions around 1/2 to 2/3 seem to work well in practice.
 */
#define USABLE_FRACTION(n) (((n) << 1)/3)
```

事实上就是2/3*size，在申请前面我们提到的柔性数组内存时，总共有三部分，`PyObjectsObject`结构体（定长）以及后面的柔性数组（包括索引数组和entry数组）。

```c
dk = PyObject_MALLOC(sizeof(PyDictKeysObject)
                             + es * size
                             + sizeof(PyDictKeyEntry) * usable);
```

再来康康`new_dict`：

```c
/* Consumes a reference to the keys object */
static PyObject *
new_dict(PyDictKeysObject *keys, PyObject **values)
{
    PyDictObject *mp;
    assert(keys != NULL);
    if (numfree) {
        //缓冲池是否有dict可以使用
        mp = free_list[--numfree];
        assert (mp != NULL);
        assert (Py_TYPE(mp) == &PyDict_Type);
        _Py_NewReference((PyObject *)mp);
    }
    else {
        mp = PyObject_GC_New(PyDictObject, &PyDict_Type);
        if (mp == NULL) {
            DK_DECREF(keys);
            free_values(values);
            return NULL;
        }
    }
    mp->ma_keys = keys;
    mp->ma_values = values;
    mp->ma_used = 0;
    mp->ma_version_tag = DICT_NEXT_VERSION();
    assert(_PyDict_CheckConsistency(mp));
    return (PyObject *)mp;
}
```

就是用`ma_keys`和`ma_values`构造一个dict，由于我们创建的字典是`combined`类型，所以传入的`ma_values`为NULL。

##### 查找

再回过头来看看`PyDictKeysObject`中的`dk_lookup`：

```c
...
/* Function to lookup in the hash table (dk_indices):

       - lookdict(): general-purpose, and may return DKIX_ERROR if (and
         only if) a comparison raises an exception.

       - lookdict_unicode(): specialized to Unicode string keys, comparison of
         which can never raise an exception; that function can never return
         DKIX_ERROR.

       - lookdict_unicode_nodummy(): similar to lookdict_unicode() but further
         specialized for Unicode string keys that cannot be the <dummy> value.

       - lookdict_split(): Version of lookdict() for split tables. */
    dict_lookup_func dk_lookup;
...
```

因为`lookdict_unicode`和`lookdict_unicode_nodummy`都是specialized的，所以我们来康康general-purpose的`lookdict`。

```c
/*
lookdict是一个广泛适用的查找函数，并且python的实现为string提供了两个特化的版本
lookdict_unicode() 和 lookdict_unicode_nodummy()
*/
static Py_ssize_t _Py_HOT_FUNCTION
lookdict(PyDictObject *mp, PyObject *key,
         Py_hash_t hash, PyObject **value_addr)
{
    size_t i, mask, perturb;
    PyDictKeysObject *dk;
    PyDictKeyEntry *ep0;

top:
    dk = mp->ma_keys;
    ep0 = DK_ENTRIES(dk);
    mask = DK_MASK(dk);
    perturb = hash;
    /* 通过与操作使得hash值落在 entry的数量之下 */
    i = (size_t)hash & mask;

    for (;;) {
        Py_ssize_t ix = dk_get_index(dk, i);
        if (ix == DKIX_EMPTY) {
            *value_addr = NULL;
            return ix;
        }
        if (ix >= 0) {
            PyDictKeyEntry *ep = &ep0[ix];
            assert(ep->me_key != NULL);
            /* 如果指向同一块内存，那么必然相等 */
            if (ep->me_key == key) {
                *value_addr = ep->me_value;
                return ix;
            }
            /* hash值相同则还要判断key值是否相等 */
            if (ep->me_hash == hash) {
                PyObject *startkey = ep->me_key;
                Py_INCREF(startkey);
                int cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
                Py_DECREF(startkey);
                if (cmp < 0) {
                    *value_addr = NULL;
                    return DKIX_ERROR;
                }
                if (dk == mp->ma_keys && ep->me_key == startkey) {
                    if (cmp > 0) {
                        *value_addr = ep->me_value;
                        return ix;
                    }
                }
                else {
                    /* The dict was mutated, restart */
                    // 这里逻辑没看懂
                    goto top;
                }
            }
        }
        perturb >>= PERTURB_SHIFT;
        i = (i*5 + perturb + 1) & mask;
    }
    Py_UNREACHABLE();
}
```

传入的有key和hash值，首先为了将hash值限制在索引数组数量之下，先做了一个与操作。接着，根据与操作之后的hash值拿到一个entry

- 首先比较entry的`me_key`和传入的key是否指向同一块内存，是的话搜索成功（比如小整数池）
- 否则比较与操作之前的hash值，相同的话再次比较key和`me_key`的值是否相同（注意和之前的地址比较区分开，这里比较的是地址中的值）

若搜索成功，会把entry的地址扔进`value_addr`中（which is a 指针的指针）。

##### 插入

康康PyDict_SetItem：

```c
/* CAUTION: PyDict_SetItem() must guarantee that it won't resize the
 * dictionary if it's merely replacing the value for an existing key.
 * This means that it's safe to loop over a dictionary with PyDict_Next()
 * and occasionally replace a value -- but you can't insert new keys or
 * remove them.
 * 这个函数只是把hash值算出来，然后调用insertdict()
 */
int
PyDict_SetItem(PyObject *op, PyObject *key, PyObject *value)
{
    PyDictObject *mp;
    Py_hash_t hash;
    if (!PyDict_Check(op)) {
        PyErr_BadInternalCall();
        return -1;
    }
    assert(key);
    assert(value);
    mp = (PyDictObject *)op;
    if (!PyUnicode_CheckExact(key) ||
        (hash = ((PyASCIIObject *) key)->hash) == -1)
    {
        hash = PyObject_Hash(key);
        if (hash == -1)
            return -1;
    }

    /* insertdict() handles any resizing that might be necessary */
    return insertdict(mp, key, hash, value);
}
```

可以看到PyDict_SetItem只是把hash值算出来，然后调用insertdict：

```c
/*
Internal routine to insert a new item into the table.
Used both by the internal resize routine and by the public insert routine.
Returns -1 if an error occurred, or 0 on success.
好像终于找到了
*/
static int
insertdict(PyDictObject *mp, PyObject *key, Py_hash_t hash, PyObject *value)
{
    PyObject *old_value;
    PyDictKeyEntry *ep;

    Py_INCREF(key);
    Py_INCREF(value);
    /* 当ma_values不为空，证明这个map是一个split table 
       所以无法共享了，调用resize重做
       这里其实应该换成 _PyDict_HasSplitTable
    */
    if (mp->ma_values != NULL && !(key)) {
        if (insertion_resize(mp) < 0)
            goto Fail;
    }

    /* 把旧值找到 */
    Py_ssize_t ix = mp->ma_keys->dk_lookup(mp, key, hash, &old_value);
    if (ix == DKIX_ERROR)
        goto Fail;

    assert(PyUnicode_CheckExact(key) || mp->ma_keys->dk_lookup == lookdict);
    MAINTAIN_TRACKING(mp, key, value);

    /* When insertion order is different from shared key, we can't share
     * the key anymore.  Convert this instance to combine table.
     */
    if (_PyDict_HasSplitTable(mp) &&
        ((ix >= 0 && old_value == NULL && mp->ma_used != ix) ||
         (ix == DKIX_EMPTY && mp->ma_used != mp->ma_keys->dk_nentries))) {
        if (insertion_resize(mp) < 0)
            goto Fail;
        ix = DKIX_EMPTY;
    }

    if (ix == DKIX_EMPTY) {
        /* Insert into new slot. */
        ...// 该key值未进行插入，见下面分析
        return 0;
    }
	
    //该key值已进行插入，见下面分析
    ...
    return 0;

Fail:
    Py_DECREF(value);
    Py_DECREF(key);
    return -1;
}
```

插入值分两种情况：

- key值未使用过
- key值已使用过

###### 当前值还未插入过

当发现插入的key值还没有被插入过时：

```c
if (ix == DKIX_EMPTY) {
        /* Insert into new slot. */
        assert(old_value == NULL);
        if (mp->ma_keys->dk_usable <= 0) {
            /* Need to resize. */
            if (insertion_resize(mp) < 0)
                goto Fail;
        }
    	// 找到可插入的哈希索引
        Py_ssize_t hashpos = find_empty_slot(mp->ma_keys, hash);
    	//找到插入entry的位置
        ep = &DK_ENTRIES(mp->ma_keys)[mp->ma_keys->dk_nentries];
    	//真实的操作在这里
        dk_set_index(mp->ma_keys, hashpos, mp->ma_keys->dk_nentries);
        ep->me_key = key;
        ep->me_hash = hash;
        /*
         * 如果是split表要把value写到ma_values中
         * 否则key和value都是绑定在一个entry中的
         */
        if (mp->ma_values) {
            assert (mp->ma_values[mp->ma_keys->dk_nentries] == NULL);
            mp->ma_values[mp->ma_keys->dk_nentries] = value;
        }
        else {
            ep->me_value = value;
        }
        mp->ma_used++;
        mp->ma_version_tag = DICT_NEXT_VERSION();
        mp->ma_keys->dk_usable--; //可用--
        mp->ma_keys->dk_nentries++;//entry往后拉
        assert(mp->ma_keys->dk_usable >= 0);
        assert(_PyDict_CheckConsistency(mp));
        return 0;
    }
```

插入一个entry的逻辑大致是：

- 通过`find_empty_slot`找到合适的索引`hash_pos`（这里会用到哈希函数，hash_pos是相对dk_indices数组而言）
- 找到当前空闲的entry数组的位置（取的是最后一个即`ep = &DK_ENTRIES(mp->ma_keys)[mp->ma_keys->dk_nentries]`）
- 通过`dk_set_index(mp->ma_keys, hashpos, mp->ma_keys->dk_nentries)`让索引和entry真实存储的位置建立关系
- 将key和value写入entry数组，这里需要根据表的类型（combined / split）进行逻辑分发
- 后续处理

那我们再看看`dk_set_index`：

```c
/* write to indices. */
/* 把ma_keys的索引和后面的entries绑定起来 */
static inline void
dk_set_index(PyDictKeysObject *keys, Py_ssize_t i, Py_ssize_t ix)
{
    Py_ssize_t s = DK_SIZE(keys);

    assert(ix >= DKIX_DUMMY);

    if (s <= 0xff) {
        int8_t *indices = (int8_t*)(keys->dk_indices);
        assert(ix <= 0x7f);
        indices[i] = (char)ix;
    }
    else if (s <= 0xffff) {
        int16_t *indices = (int16_t*)(keys->dk_indices);
        assert(ix <= 0x7fff);
        indices[i] = (int16_t)ix;
    }
#if SIZEOF_VOID_P > 4
    else if (s > 0xffffffff) {
        int64_t *indices = (int64_t*)(keys->dk_indices);
        indices[i] = ix;
    }
#endif
    else {
        int32_t *indices = (int32_t*)(keys->dk_indices);
        assert(ix <= 0x7fffffff);
        indices[i] = (int32_t)ix;
    }
}
```

逻辑比较简单，麻烦的一点是需要根据索引数组元素的大小去对指针做强制转换。

###### 当前值已经插入过

如果已经插入过该key，无论是Dummy状态还是Active态，直接扔进entries数组即可。

```c
if (_PyDict_HasSplitTable(mp)) {
        mp->ma_values[ix] = value;
        if (old_value == NULL) {
            /* pending state */
            assert(ix == mp->ma_used);
            mp->ma_used++;
        }
    }
    else {
        assert(old_value != NULL);
        DK_ENTRIES(mp->ma_keys)[ix].me_value = value;
    }

    mp->ma_version_tag = DICT_NEXT_VERSION();
    Py_XDECREF(old_value); /* which **CAN** re-enter (see issue #22653) */
    assert(_PyDict_CheckConsistency(mp));
    Py_DECREF(key);
    return 0;
```

#### 2和3的一些区别

这里简单说说2和3中字典是线上的一些区别，首先是加入了字典类型的概念（split和combined），在上面的解析中，因为我们是基本只走combined类型字典的这条线索，所以可能忽略了许多不知所云的代码，因为在一些情况下split表需要向combined表进行转换，这里由于我没有深入了解也略过不表。

此外，事实上在2的实现中，会有一个固定大小（一般为8）的小hash表`ma_smalltable`，当元素数量较少时不用动态申请内存。有兴趣的可以看看[这里]( https://github.com/python/cpython/blob/2.7/Include/dictobject.h#L88 )

```c
struct _dictobject {
    PyObject_HEAD
    Py_ssize_t ma_fill;  /* # Active + # Dummy */
    Py_ssize_t ma_used;  /* # Active */

    /* The table contains ma_mask + 1 slots, and that's a power of 2.
     * We store the mask instead of the size because the mask is more
     * frequently needed.
     */
    Py_ssize_t ma_mask;

    /* ma_table points to ma_smalltable for small tables, else to
     * additional malloc'ed memory.  ma_table is never NULL!  This rule
     * saves repeated runtime null-tests in the workhorse getitem and
     * setitem calls.
     */
    PyDictEntry *ma_table;
    PyDictEntry *(*ma_lookup)(PyDictObject *mp, PyObject *key, long hash);
    PyDictEntry ma_smalltable[PyDict_MINSIZE];
};
```

抛弃小表的原因大概率是加入了字典类型的概念后，如果遵循该实现会比较之复杂。

#### 总结

零零散散写了几天，虽然一直在摸鱼，在家是真的爽~ 成绩也快出完了... 希望能好好过年--