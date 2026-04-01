# .gitignore 忽略文件
{docsify-updated}

.gitignore 只对“尚未被 Git 跟踪的文件”生效，对已经纳入版本控制的文件完全无效。

`.gitignore` 每一行一条规则：
1. 忽略某个文件 : `a.txt`
2. 忽略某个目录，末尾 / 表示目录 : `target/`
3. 通配符
   1. `*` : 任意多个字符（不含 /）
   2. `?` : 任意单个字符
   3. `**` : 任意层级目录
4. 指定路径（相对仓库根目录） : `/src/main/resources/app.yml`
5. 反向匹配（不忽略），用 `!` 表示例外 :  `!important.log`
6. 不以 `/` 开头 → 任意层级匹配 :  `config.yml` 会忽略 `config.yml` , `a/config.yml` ,  `a/b/config.yml`
