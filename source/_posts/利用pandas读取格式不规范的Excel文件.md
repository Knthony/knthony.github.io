---
title:  利用pandas读取格式不规范的Excel文件
date: 2020/9/12 18:14:33
categories:
- Tech
tags:
- pandas
- excel
---

![excel_range_header_3](https://pbpython.com/images/excel_range_header_3.png)



## 介绍

`pandas` 很容易将Excel文件读取为`DataFrame`，但是在现实中，Excel文件里面的数据格式往往是不规范的，在那些数据分散在不同Sheet的情况下，就需要自定义读取数据的方式，这篇文章将讨论如何用`pandas`和`openpyxl`读取这类格式的Excel文件，将里面的数据转换为`DataFrame`以便进一步的分析工作。



## 数据的问题

`pandas`内的`read_excel`方法在读取Excel工作表方面非常高效好用，无论如何，当数据在表中不是以连续的形式存储的话，读取出来的数据可能就和预期的不同了。



当你尝试用`read_excel`读取下面图中所示的这种数据格式时：

![excel_ranges](https://pbpython.com/images/excel_ranges.png)



你将得到如下结果：

![excel_range_dataframe](https://pbpython.com/images/excel_range_dataframe.png)

上面的结果包含了很多`Unnamed` 的列。



## Pandas 解决方案

最简单的方案

此数据集的最简单解决方案是在`read_excel（）`方法中使用`header`和`usecols`参数，特别是`usecols`对于控制想要提取的数据列很有用。

这些例子的所有文件都在[github](https://github.com/chris1610/pbpython/blob/master/data/shipping_tables.xlsx)

下面是一种我们提取数据的方法：



```python
import pandas as pd
from pathlib import Path
src_file = Path.cwd() /  'shipping_tables.xlsx'

df = pd.read_excel(src_file, header=1, usecols='B:F')
```

下面是得到的结果



![excel_range_dataframe_clean](https://pbpython.com/images/excel_range_dataframe_clean.png)



逻辑很简单，`usecols`参数接受Excel文件中的列范围，比如（B:F）,表示程序只读取这个范围内的数据，`header`参数需要一个用于定义标题列的整数。1表示Excel中的第二行。



在一些实例中，我们可能想用数字列表来表示所要提取数据的列：

```python
df = pd.read_excel(src_file, header=1, usecols=[1,2,3,4,5])
```

如果在对大型数据集使用这种数字模式（每三列，或仅偶数列）那这个方法就很有用。

`usecols`还可以使用列名来表示，如下：

```python
df = pd.read_excel(
    src_file,
    header=1,
    usecols=['item_type', 'order id', 'order date', 'state', 'priority'])
```

如果确认这些列名不会改变，那是用上面的方法实现也很方便。

最后，`usecols`的高级用法，回调函数，下面的例子实现了将`unnamed`和`priority`过滤掉的功能

```python
# Define a more complex function:
def column_check(x):
    if 'unnamed' in x.lower():
        return False
    if 'priority' in x.lower():
        return False
    if 'order' in x.lower():
        return True
    return True

df = pd.read_excel(src_file, header=1, usecols=column_check)
```

使用回调函数的另一个方法是用`lambda`表达式，通过判断列名是否在我们的定义好的列表中。

```python
cols_to_use = ['item_type', 'order id', 'order date', 'state', 'priority']
df = pd.read_excel(src_file,
                   header=1,
                   usecols=lambda x: x.lower() in cols_to_use)
```

回调函数给了我们灵活的方式去处理真实世界里的Excel文件



## 范围和表格

在一些情况中，Excel表中的数据会很混乱，举个例子，我们有一个表叫`ship_cost`我们想把它读取出来。



![excel_named_table-2](https://pbpython.com/images/excel_named_table-2.png)

在这里，我们可以直接使用`openpyxl`来解析文件将数据转换为`DataFrame`，这样处理Excel实际上更容易点。

```python
from openpyxl import load_workbook
import pandas as pd
from pathlib import Path
src_file = src_file = Path.cwd() / 'shipping_tables.xlsx'

wb = load_workbook(filename = src_file)
```

上面加在了所有的`worksheet`，如果你想看所有的`sheet`

`wb.sheetnames`

`['sales', 'shipping_rates']`

访问指定的列

`sheet = wb['shipping_rates']`

列出所有的表明

`sheet.tables.keys()`

`dict_keys(['ship_cost'])`

该键对应于我们在Excel中分配给表的名称。现在我们访问表以获取等效的Excel范围

```python
lookup_table = sheet.tables['ship_cost']
lookup_table.ref
```

`'C8:E16'`

现在我们知道了我们加载的数据范围，最后一步是将它转换为DataFrame格式。

```python
# Access the data in the table range
data = sheet[lookup_table.ref]
rows_list = []

# Loop through each row and get the values in the cells
for row in data:
    # Get a list of all columns in each row
    cols = []
    for col in row:
        cols.append(col.value)
    rows_list.append(cols)

# Create a pandas dataframe from the rows_list.
# The first row is the column names
df = pd.DataFrame(data=rows_list[1:], index=None, columns=rows_list[0])
```

下面是结果

![excel_shipping_dataframe](https://pbpython.com/images/excel_shipping_dataframe.png)

现在，我们有一个干净整齐的表以便我们后面的分析操作了！



> 此文章翻译自https://pbpython.com/pandas-excel-range.html