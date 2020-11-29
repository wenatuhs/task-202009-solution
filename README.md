# task-202009-solution

## 文件说明

- 我使用 Python 在 Jupyter Lab 环境中做了一个实现，具体过程及代码请参见 *task-202009-solution.ipynb* 文件
- 该深圳 A 股切片的每一支股票的预测值的 `r_squared` 请参见 *r_squared.csv* 文件
- 每支股票 log 中的无效数据都保存在其对应的 `invalid` 字段中，请参见 *final.ckpt* 文件
    - *final.ckpt* 文件可用 Python 的 *pickle* 库读取

## 实现思路

### 数据结构

首先：

```
    r_squared = 1 - res_ss / tot_ss = 1 - res_ms / (y_ms - y_m ** 2)
```

其中 `res_ss` 是 residual sum of squares，`tot_ss` 是 total sum of squares，`res_ms` 是 residuals mean of squares，`y_ms` 是真实值 y 的 mean of squares，`y_m` 是真实值 y 的 mean。

这种形式意味着各统计量可以进行动态更新而无需保留过多的历史状态：

```
    y_m' = n / (n + 1) * y_m + 1 / (n + 1) * y
    y_ms' = n / (n + 1) * y_ms + 1 / (n + 1) * y ** 2
    res_ms' = n / (n + 1) * res_ms + 1 / (n + 1) * (y - f) ** 2
```

因此设计了下面的数据结构：

```python
stock = {
    'count': 0,  # number of the full-info points
    'y_m': 0,  # mean of the real values
    'y_ms': 0,  # mean square of the real values
    'res_ms': 0,  # mean square of the residuals
    'cache': [],  # cached market/prediction data of the last n seconds
    'invalid': [],  # optional, store the invalid points that removed from cache
    ...
}
```

其中 `cache` 和 `invalid` 中保存的是 `record` 序列，`record` 结构如下：

```python
record = {
    'idx': None,  # data index, the number after '#' or '##'
    't': None,  # ETS in second
    'f': None,  # prediction
    'mt': None,  # current mid-value
    'mtn': None  # mid-value after around n seconds
}
```

注意一个 `record` 一般由一条 market log 或者一条 prediction log 创建：

- 如果由 market log 创建，则 `idx`，`t` 以及 `mt` 字段被填充，其他字段依然为 `None`
- 如果由 prediction log 创建，则 `idx` 以及 `f` 字段被填充，其他字段依然为 `None`

每条 `record` 可能在随后的 log 解析中被多次更新。

**注意：由于题干中给定 `ETS` 为当前时间，而 `ETS` 的格式只有时分秒和毫秒，所以在整个数据处理过程中我们假定数据不会跨日期！（后续也验证了全部数据都在同一天）**

由于有多支股票（2126 支），整体的数据使用字典保存，结构为：

```python
stocks_stats = {
    '000007.SZ': stock_1,
    '000008.SZ': stock_2,
    ...
}
```

当数据收集完成后，对每一个 `stock`，使用下面公式即可算出 `r_squared`：

```
    r_squared = 1 - res_ms / (y_ms - y_m ** 2)
```

### 数据流

每个 blob 中的每一行 log 进行 parse 后，更新 `stocks_stats`；为计算真值，每支股票需要保留至少 5s 的历史记录（cache）。我们也希望能够保存所有异常数据，留待后续分析。所以整体的 `stocks_stats` 更新流程如下：

1. 解析一行 log 并更新对应 record
    - 如果是 market log，尝试找到其对应的 prediction 记录并更新 record。如果找到，更新 record，回溯 `cache`，看本条记录的 `mt` 是否能够成为 `cache` 中最早 records 的 `mtn`，如果是，更新较早 records 的 `mtn`
    - 如果是 prediction log，尝试找到其对应的 market 记录并更新 record
2. 消除 `cache` 中的头部无效数据或完整数据，更新统计结果
    - 如果是无效数据，pop 该数据并压入 `invalid` 列表
    - 如果是完整数据（即已获得预测值 f 并可以计算真值 y 的数据），计算相应统计量，更新 `y_m`，`y_ms` 以及 `res_ms`，随后 pop 该数据

通过以上的数据更新方式，我们可以保证每支股票的 `cache` 最多只存 5s 的 records，且一切异常数据都进入 `invalid` 列表。这样程序所用内存可以达到最优（不保存任何过期数据）。当内存需要进一步优化时，可以去掉 `invalid` 相关的逻辑，内存使用可以更接近极限。

**注意：更新 `mtn` 时我们采用相对原始点最接近 5s 间隔的点，但是为保证结果的可靠性，我们使用了 1s 的阈值，也即只有相对原始点时间间隔 4s 到 6s 之内的点才算有效点。如果找不到这样的有效点，则原始点被认为是无效记录。**

### 数据异常处理

数据中的异常主要有三类：

- `A1 - B1 <= 0`，这类数据的 `mt` 为 `nan`
- 数据跳跃，也就是有 `#` 编号所对应的预测值，但是没有对应的市场数据，这类数据的 `t` 和 `mt` 为 `nan`
- 无可靠 `n`-sec 数据，因此无法计算真值，这类数据的 `mtn` 为 `nan`

所有异常数据都被存在对应 `stock` 的 `invalid` 列表中。**最终的 `r_squared` 统计结果只包含有效数据。**
