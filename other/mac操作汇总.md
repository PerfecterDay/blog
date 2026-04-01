# MacOs 

## 启用和关闭 root 用户
MacOs 默认是关闭 root 用户的，启用使用 `dsenableroot` ， 关闭 `dsenableroot -d` 。

### 用户与群组高级选项
按住 `control` 在用户名上单击就会展示：
<center><img src="pics/macos-user.png" width="40%"></center>


## 提示文件损坏的解决方案
`sudo xattr -rd com.apple.quarantine /Applications/Xournal++.app`