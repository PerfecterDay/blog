# PromQL 函数
{docsify-updated}

以下函数只能作用在范围向量上：
+ increase(range-vector)： 一段时间范围内的增长量
+ changes(range-vector)：
+ absent_over_time(range-vector)
+ delta(range-vector)
+ deriv(range-vector)
+ holt_winters(range-vector, scalar, scalar)
+ idelta(range-vector)
+ irate(range-vector)
+ predict_linear(range-vector, scalar)
+ rate(range-vector)
+ resets(range-vector)
+ avg_over_time(range-vector)
+ min_over_time(range-vector)
+ max_over_time(range-vector)
+ sum_over_time(range-vector)
+ count_over_time(range-vector)
+ quantile_over_time(scalar, range-vector)
+ stddev_over_time(range-vector)
+ stdvar_over_time(range-vector)

所有的这些函数都会返回一个瞬时向量以可以用图表的形式绘制出来。