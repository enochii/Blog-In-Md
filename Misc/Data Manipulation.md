#### fastjson

前端的JSONArray转换后里面的每个item是一个(Linked)HashMap（为啥不是JSONObject）...



#### Java Stream

转Stream后使用一次就会失效。

`flatMap`

>http://dblab.xmu.edu.cn/blog/961-2/ 

用`flatMap`可以伪造`join`

> 去水了下SO：https://stackoverflow.com/a/59243531/10701129



#### pandas

pandas的join有按index(join)和按column(merge)

> https://stackoverflow.com/questions/28228781/why-does-pandas-inner-join-give-valueerror-lenleft-on-must-equal-the-number-o 



---

| ECGI_1 | Freq_1 | CellID_1 | ECGI_2 | Freq_2 | CellID_2 | ECGI_3 |  Freq_3 | CellID_3 |
| ------ | ----- | ------ | -------- | ------ | --------- | ----- | ------ | -------- |
|          |       |        |          |        |           |       |        |          |

有以上格式的DataFrame，ECGI_i，Freq_i和CellID_i可以共同确定一个item（i∈[1, 8)），需要做的是提取出所有互异且满足条件的item，手写了个for，数据有1.5w+行，花了快20s...

```python
for index, row in traj.iterrows():
    #     print(row)
    for i in range(1, 8):
        COLUMN1_NAME = Traj_manager.PREFIX_ECGI + str(i)
        COLUMN2_NAME = Traj_manager.PREFIX_Freq + str(i)
        COLUMN3_NAME = Traj_manager.PREFIX_CellID + str(i)
        #         print(row[COLUMN_NAME])
        if row[COLUMN1_NAME] == 999 :
            break
        ECGI_ids.add(
            tuple([row[COLUMN1_NAME], row[COLUMN2_NAME], row[COLUMN3_NAME]])
        )
```

合理一点应该用`DataFrame`内置的操作，思考后解决方案如下：

```python
	ECGI_all = pd.DataFrame()
    for i in range(1, 8):
        COLUMN1_NAME = Traj_manager.PREFIX_ECGI + str(i)
        COLUMN2_NAME = Traj_manager.PREFIX_Freq + str(i)
        COLUMN3_NAME = Traj_manager.PREFIX_CellID + str(i)
        
        ECGI_all = ECGI_all.append(traj[[COLUMN1_NAME, COLUMN2_NAME, COLUMN3_NAME]].rename(columns={COLUMN1_NAME : 'k1',
                              COLUMN2_NAME : 'k2',
                              COLUMN3_NAME : 'k3'
                              }).drop_duplicates())

    # print(ECGI_all)
    ECGI_all = ECGI_all.drop_duplicates()
    # filter operation
    ECGI_ids = ECGI_all[ECGI_all['k1'] != -999]
    ret = zip(ECGI_ids['k1'].to_list(),
              ECGI_ids['k2'].to_list(),
              ECGI_ids['k3'].to_list()
              )

    return list(ret)
```

也就是先按列（1-7）把三个id取出来，rename后再append一下，最后做一下filter，差不多3-4s可以完成，大概是**单指令多数据流**的原因。这里要注意append会返回一个新的DataFrame。

这里用到了python的`zip`操作，因为直接将DataFrame `to_dict`之后，格式大概是：

```shell
{
	k1:[],
	k2:[],
	k3:[]
}
```

我希望拿到的是：

```shell
{
	[
		[k11, k12, k13],
		...
	]
}
```

用`zip`转一下，另外最后传成`json`，zip好像因为内部用了`yield`的原因不是可序列化的，做一下list转换（及时求值？）就ok了。



> https://stackoverflow.com/questions/27370731/pandas-how-to-generate-multiple-rows-by-one-row/59288941#59288941 
>
> https://stackoverflow.com/a/59288941/10701129
>
> 感觉上一个问题也可以用`stack()`来做，找时间康康--



pandas读取`csv`文件显示`MemoryError`，看了一圈好像是内存占用过大，尝试指定`chunksize`要搞一堆神奇的操作，遂删除了一些无用的列，问题解决。