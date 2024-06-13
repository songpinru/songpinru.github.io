# 数据结构

## Series

Series是带有索引的一维数组,数组内的数据类型不需要相同,且索引可以重复，可以视为一行或者一列数据.
使用方式同数组(numpy数组亦可)

**生成Series**

Series可以由以下数据结构生成:
* 数组(一维数组,如果data是多维数组,里层的数组会被视为值)
* 字典(key会成为索引,value会成为值)
* 标量(标量值)

```python
# 数组
pd.Series(np.random.randn(5), index=['a', 'b', 'c', 'd', 'e'])

# 字典
pd.Series({'a': 0., 'b': 1., 'c': 2.})

# 标量
pd.Series(5., index=['a', 'b', 'c', 'd', 'e'])

```

## DataFrame

DataFrame是含有索引和列的二维数组,可视为由多个相同索引的Series组成(可以是列,也可以是行,如果series的索引是列名,则Series是一行数据)

**生成DataFrame**

DataFrame可以由以下数据结构生成:
* 数组(数组内的数据按行分布)
  * 一维数组(值为:字典,Series,列表,ndarray,元组)
  * 二维数组(一维数据为行,二维数据为列)
* 字典
  * key为列名,value为一列数据,类型可以是:数组,Series,列表,ndarray

```python
# 数组
data = [['Alex',10],['Bob',12],['Clarke',13]]
df = pd.DataFrame(data,columns=['Name','Age'],dtype=float) # 将第一维度数据转为为行，第二维度数据转化为列，即 3 行 2 列，并设置列标签

data = [{'a': 1, 'b': 2},{'a': 5, 'b': 10, 'c': 20}] # 列表对应的是第一维，即行，字典为同一行不同列元素
df = pd.DataFrame(data) # 第 1 行 3 列没有元素，自动添加 NaN (Not a Number)

# 字典
data = {'Name':['Tom', 'Jack', 'Steve', 'Ricky'],'Age':[28,34,29,42]} # 两组列元素，并且个数需要相同
df = pd.DataFrame(data) # 这里默认的 index 就是 range(n)，n 是列表的长度

# index 与序列长度相投
# 字典不同的 key 代表一个列的表头，pd.Series 作为 value 作为该列的元素
d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']),
     'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}
df = pd.DataFrame(d)
```

# 操作数据

Series的操作类似数组和列表,支持切片

DataFrame的操作类似字典和二维数组

loc和iloc



