# markdown语法
{docsify-updated}

1. 链接：`[链接名称](链接地址)`
2. 引用：`>`


### 数学公式

+ 行内公式：$A=x_1^2+x_2^2$
+ 行间公式：$$\sqrt{3} $$
+ 公式换行：$$T_0 = 0;\\ T_n = 2T_(n-1)+1 $$

+ 上标符号，符号：^，如：$x^4$
+ 下标符号，符号：_，如：$x_1$
+ 组合符号，符号：{}，如：${16}_{8}O{2+}_{2}$

+ 等于运算，符号：=，如：$x+y=z$
+ 大于运算，符号：>，如：$x+y>z$
+ 小于运算，符号：<，如：$x+y<z$
+ 大于等于运算，符号：\geq，如：$x+y \geq z$
+ 小于等于运算，符号：\leq，如：$x+y \leq z$
+ 不等于运算，符号：\neq，如：$x+y \neq z$
+ 不大于等于运算，符号：\ngeq，如：$x+y \ngeq z$
+ 不大于等于运算，符号：\not\geq，如：$x+y \not\geq z$
+ 不小于等于运算，符号：\nleq，如：$x+y \nleq z$
+ 不小于等于运算，符号：\not\leq，如：$x+y \not\leq z$
+ 约等于运算，符号：\approx，如：$x+y \approx z$
+ 恒定等于运算，符号：\equiv，如：$x+y \equiv z$

+ 分式表示，符号：{分子} \voer {分母}，如：${x+y} \over {y+z}$

### 表格
要添加表，请使用三个或多个连字符（---）创建每列的标题，并使用管道（|）分隔每列。您可以选择在表的任一端添加管道。  
可以通过在标题行中的连字符的左侧，右侧或两侧添加冒号（:），将列中的文本对齐到左侧，右侧或中心。
```
| Syntax      | Description | Test Text     |
| :---        |    :----:   |          ---: |
| Header      | Title       | Here's this   |
| Paragraph   | Text        | And more      |
```

Latex支持：https://katex.org/docs/supported.html