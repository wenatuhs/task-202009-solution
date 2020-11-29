# task-202009-solution

我使用 Python 在 Jupyter Lab 环境中做了一个实现，具体过程及代码请参见 *task-202009-solution.ipynb* 文件。该深圳 A 股切片的每一支股票的预测值的 `r_squared` 请参见 *r_squared.csv* 文件。

## 实现思路

首先：

```
    r_squared = 1 - res_ss / tot_ss = 1 - res_ms / (y_ms - y_m ** 2)
```
其中 `res_ss` 是 residuals sum of squares，`tot_ss` 是 total sum of squares，`res_ms` 是 residuals mean of squares，`y_ms` 是真实值 y 的 mean of squares，`y_m` 是真实值 y 的 mean。
