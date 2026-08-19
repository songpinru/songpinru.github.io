---
title: "Pandas"
description: "Pandas 入门指南、数据结构、基础操作与 SQL 比较"
---


## 十分钟入门 Pandas

本节帮助 Pandas 新手快速上手。导入 Pandas 与 NumPy：

```python
import numpy as np
import pandas as pd
```

### 生成对象

用值列表生成 Series，Pandas 默认自动生成整数索引：

```python
s = pd.Series([1, 3, 5, np.nan, 6, 8])
```

用含日期时间索引与标签的 NumPy 数组生成 DataFrame：

```python
dates = pd.date_range('20130101', periods=6)
df = pd.DataFrame(np.random.randn(6, 4), index=dates, columns=list('ABCD'))
```

用 Series 字典对象生成 DataFrame：

```python
df2 = pd.DataFrame({ 'A': 1., 'B': pd.Timestamp('20130102'),
    'C': pd.Series(1, index=list(range(4)), dtype='float32'),
    'D': np.array([3] * 4, dtype='int32'),
    'E': pd.Categorical(["test", "train", "test", "train"]), 'F': 'foo' })
df2.dtypes
```

### 查看数据

```python
df.head()              # 头部数据，默认 5 行
df.tail(3)             # 尾部数据
df.index               # 索引
df.columns             # 列名
df.to_numpy()          # 底层 NumPy 数组
df.describe()          # 统计摘要
df.T                   # 转置
df.sort_index(axis=1, ascending=False)  # 按轴排序
df.sort_values(by='B')                  # 按值排序
```

### 选择

```python
df['A']                              # 选择单列
df[0:3]                              # 行切片
df.loc[dates[0]]                     # 按标签提取行
df.loc[:, ['A', 'B']]                # 选择多列
df.loc['20130102':'20130104', ['A', 'B']]  # 标签切片
df.iloc[3]                           # 按位置选择
df.iloc[3:5, 0:2]                    # 整数切片
df.iloc[[1, 2, 4], [0, 2]]           # 整数列表
df[df.A > 0]                         # 布尔索引
df[df > 0]                           # 条件筛选
df2[df2['E'].isin(['two', 'four'])]  # isin 筛选
```

### 赋值

```python
df['F'] = s1                         # 新增列
df.at[dates[0], 'A'] = 0             # 按标签赋值
df.iat[0, 1] = 0                     # 按位置赋值
df.loc[:, 'D'] = np.array([5] * len(df))
df2[df2 > 0] = -df2                  # 条件赋值
```

### 缺失值

```python
df1 = df.reindex(index=dates[0:4], columns=list(df.columns) + ['E'])
df1.dropna(how='any')                # 删除含缺失值的行
df1.fillna(value=5)                  # 填充缺失值
pd.isna(df1)                         # 布尔掩码
```

### 运算

```python
df.mean()                            # 每列均值
df.mean(1)                           # 每行均值
df.apply(np.cumsum)                  # Apply 函数
df.apply(lambda x: x.max() - x.min())
s.value_counts()                     # 直方图
s.str.lower()                        # 字符串方法
```

### 合并

```python
pd.concat([df[:3], df[3:7], df[7:]])  # Concat
pd.merge(left, right, on='key')        # Join
```

### 分组

```python
df.groupby('A').sum()
df.groupby(['A', 'B']).sum()
```

### 重塑

```python
stacked = df2.stack()
stacked.unstack()
```

### 数据透视表

```python
pd.pivot_table(df, values='D', index=['A', 'B'], columns=['C'])
```

### 时间序列

```python
ts.resample('5Min').sum()            # 重采样
ts.tz_localize('UTC')                # 时区设置
ts.tz_convert('US/Eastern')          # 时区转换
ts.to_period()                       # 转时间段
```

### 类别型

```python
df["grade"] = df["raw_grade"].astype("category")
df["grade"].cat.categories = ["very good", "good", "very bad"]
```

### 可视化

```python
ts = pd.Series(np.random.randn(1000), index=pd.date_range('1/1/2000', periods=1000))
ts = ts.cumsum()
ts.plot()
```

### 数据输入 / 输出

```python
df.to_csv('foo.csv')                 # CSV
pd.read_csv('foo.csv')
df.to_hdf('foo.h5', 'df')            # HDF5
pd.read_hdf('foo.h5', 'df')
df.to_excel('foo.xlsx')              # Excel
pd.read_excel('foo.xlsx', 'Sheet1')
```

## 数据结构简介

### Series

Series 是带标签的一维数组，可存储整数、浮点数、字符串、Python 对象等类型的数据。轴标签统称为**索引**。

```python
s = pd.Series(data, index=index)
```

`data` 支持：Python 字典、多维数组、标量值。

**多维数组**

```python
s = pd.Series(np.random.randn(5), index=['a', 'b', 'c', 'd', 'e'])
s.index
# Index(['a', 'b', 'c', 'd', 'e'], dtype='object')
```

未指定 index 时，创建数值型索引 `[0, ..., len(data)-1]`。

**字典**

```python
d = {'b': 1, 'a': 0, 'c': 2}
pd.Series(d)
```

设置了 index 时，按索引标签提取值，缺失值为 `NaN`：

```python
pd.Series(d, index=['b', 'c', 'd', 'a'])
# b    1.0, c    2.0, d    NaN, a    0.0
```

**标量值**

```python
pd.Series(5., index=['a', 'b', 'c', 'd', 'e'])
```

**Series 类似多维数组**

```python
s[0]                    # 按位置索引
s[:3]                   # 切片
s[s > s.median()]       # 布尔选择
s[[4, 3, 1]]            # 数组索引
np.exp(s)               # NumPy 函数
s.array                 # 提取扩展数组
s.to_numpy()            # 提取 NumPy 数组
```

**Series 类似字典**

```python
s['a']                  # 用标签提取值
s['e'] = 12.            # 设置值
'e' in s                # 成员检查
s.get('f')              # 安全提取，不存在返回 None
```

**矢量操作与对齐**

```python
s + s                   # 矢量加法
s * 2                   # 标量乘法
s[1:] + s[:-1]          # 自动基于标签对齐，不重叠的标签结果为 NaN
```

**名称属性**

```python
s = pd.Series(np.random.randn(5), name='something')
s.name
s2 = s.rename("different")
```

### DataFrame

DataFrame 是由多种类型的列构成的二维标签数据结构，类似于 Excel 或 SQL 表。

```python
df = pd.DataFrame(data, index=index, columns=columns)
```

**用 Series 字典生成**

```python
d = {'one': pd.Series([1., 2., 3.], index=['a', 'b', 'c']),
     'two': pd.Series([1., 2., 3., 4.], index=['a', 'b', 'c', 'd'])}
df = pd.DataFrame(d)
pd.DataFrame(d, index=['d', 'b', 'a'], columns=['two', 'three'])
```

**用多维数组字典生成**

```python
d = {'one': [1., 2., 3., 4.], 'two': [4., 3., 2., 1.]}
pd.DataFrame(d)
pd.DataFrame(d, index=['a', 'b', 'c', 'd'])
```

**用列表字典生成**

```python
data2 = [{'a': 1, 'b': 2}, {'a': 5, 'b': 10, 'c': 20}]
pd.DataFrame(data2)
```

**用元组字典生成多层索引**

```python
pd.DataFrame({('a', 'b'): {('A', 'B'): 1, ('A', 'C'): 2}})
```

**提取、添加、删除列**

```python
df['one']                    # 提取列
df['three'] = df['one'] * df['two']  # 新增列
del df['two']                # 删除列
df.pop('three')              # 弹出列
df.insert(1, 'bar', df['one'])  # 指定位置插入
```

**用方法链分配新列**

```python
iris.assign(sepal_ratio=iris['SepalWidth'] / iris['SepalLength']).head()
iris.assign(sepal_ratio=lambda x: (x['SepalWidth'] / x['SepalLength'])).head()
```

**数据对齐与运算**

```python
df + df2                          # 自动对齐列与行标签
df - df.iloc[0]                   # DataFrame 与 Series 广播
df.sub(row, axis='columns')       # 显式控制广播方向
```

**转置与 NumPy 函数**

```python
df[:5].T
np.exp(df)
np.asarray(df)
```

**控制台显示**

```python
baseball.info()
pd.set_option('display.width', 40)
pd.set_option('display.max_colwidth', 30)
```

## 基础操作

### Head 与 Tail

```python
long_series.head()      # 默认 5 条
long_series.tail(3)     # 指定数量
```

### 属性与底层数据

```python
df.shape                # 形状
df.columns              # 列名
df.index                # 行索引
s.array                 # 扩展数组
s.to_numpy()            # NumPy 数组
```

### 加速操作

Pandas 可借助 `numexpr` 与 `bottleneck` 支持库加速特定类型的二进制数值与布尔操作。强烈建议安装这两个库。

### 二进制操作

```python
df.sub(row, axis='columns')   # 按列广播
df.sub(column, axis='index')  # 按行广播
df.add(df2, fill_value=0)     # 填充缺失值运算
```

比较操作：`eq, ne, lt, gt, le, ge`

布尔简化：`(df > 0).all(), (df > 0).any(), df.empty`

### 函数应用

```python
df.apply(np.cumsum)
df.apply(lambda x: x.max() - x.min())
df.applymap(lambda x: '%.2f' % x)  # 逐元素
```

### 重建索引

```python
df.reindex(index=dates[0:4], columns=list(df.columns) + ['E'])
```

### 迭代

```python
for col in df:           # 迭代列名
for idx, row in df.iterrows():  # 迭代行
for col_name, series in df.items():  # 迭代列
```

### .dt 访问器

```python
s = pd.Series(pd.date_range('20130101 09:10:12', periods=4))
s.dt.hour
s.dt.day
s.dt.tz_localize('US/Eastern')
s.dt.strftime('%Y/%m/%d')
```

### 矢量化字符串方法

```python
s = pd.Series(['A', 'B', 'C', 'Aaba', 'Baca', np.nan, 'CABA', 'dog', 'cat'])
s.str.lower()
s.str.upper()
```

### 排序

```python
df.sort_index()                    # 按索引排序
df.sort_values(by='two')           # 按值排序
df.sort_values(by=['one', 'two'])  # 多列排序
```

### 复制

```python
shallow = df.copy()               # 深复制
deep = df.copy(deep=True)
```

### 数据类型

```python
df3.astype('float32')              # 转换所有列
dft.astype({'a': np.bool, 'c': np.float64})  # 字典指定
pd.to_numeric(m)                   # 转换为数值型
pd.to_datetime(m)                  # 转换为 datetime
pd.to_timedelta(m)                 # 转换为 timedelta
pd.to_numeric(m, errors='coerce')  # 强制转换
df.select_dtypes(include=[bool])   # 基于 dtype 选择列
```

## 与 SQL 比较

本页使用 tips 数据集：

```python
url = 'https://raw.github.com/pandas-dev/pandas/master/pandas/tests/data/tips.csv'
tips = pd.read_csv(url)
```

### SELECT

```sql
SELECT total_bill, tip, smoker, time FROM tips LIMIT 5;
```

```python
tips[['total_bill', 'tip', 'smoker', 'time']].head(5)
```

### WHERE

```sql
SELECT * FROM tips WHERE time = 'Dinner' LIMIT 5;
```

```python
tips[tips['time'] == 'Dinner'].head(5)
```

多条件组合：

```sql
SELECT * FROM tips WHERE time = 'Dinner' AND tip > 5.00;
```

```python
tips[(tips['time'] == 'Dinner') & (tips['tip'] > 5.00)]
```

```sql
SELECT * FROM tips WHERE size >= 5 OR total_bill > 45;
```

```python
tips[(tips['size'] >= 5) | (tips['total_bill'] > 45)]
```

NULL 判断：

```python
frame[frame['col2'].isna()]      # IS NULL
frame[frame['col1'].notna()]     # IS NOT NULL
```

### GROUP BY

```sql
SELECT sex, count(*) FROM tips GROUP BY sex;
```

```python
tips.groupby('sex').size()
```

```sql
SELECT day, AVG(tip), COUNT(*) FROM tips GROUP BY day;
```

```python
tips.groupby('day').agg({'tip': np.mean, 'day': np.size})
```

多列分组：

```sql
SELECT smoker, day, COUNT(*), AVG(tip) FROM tips GROUP BY smoker, day;
```

```python
tips.groupby(['smoker', 'day']).agg({'tip': [np.size, np.mean]})
```

### JOIN

```python
df1 = pd.DataFrame({'key': ['A', 'B', 'C', 'D'], 'value': np.random.randn(4)})
df2 = pd.DataFrame({'key': ['B', 'D', 'D', 'E'], 'value': np.random.randn(4)})
```

```python
pd.merge(df1, df2, on='key')              # INNER JOIN
pd.merge(df1, df2, on='key', how='left')  # LEFT JOIN
pd.merge(df1, df2, on='key', how='right') # RIGHT JOIN
pd.merge(df1, df2, on='key', how='outer') # FULL JOIN
```

### UNION

```python
pd.concat([df1, df2])                     # UNION ALL
pd.concat([df1, df2]).drop_duplicates()   # UNION
```

### 排序与分页

```sql
SELECT * FROM tips ORDER BY tip DESC LIMIT 10 OFFSET 5;
```

```python
tips.nlargest(10 + 5, columns='tip').tail(10)
```

### 更新

```sql
UPDATE tips SET tip = tip*2 WHERE tip < 2;
```

```python
tips.loc[tips['tip'] < 2, 'tip'] *= 2
```

### 删除

```sql
DELETE FROM tips WHERE tip > 9;
```

```python
tips = tips.loc[tips['tip'] <= 9]
```
