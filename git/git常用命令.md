# git 常用命令
{docsify-updated}

### Github 配置 SSH 登录
1. 打开 git bash，非 windows 下不需要
2. 查看是否已经有 SSH keys： `ls -la ~/.ssh`
3. 生成 SSH keys: `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`
4. 启动 ssh-agent：<code>eval &#96;ssh-agent -s&#96;</code>
5. 添加 key 到 ssh-agent: `ssh-add ~/.ssh/id_ed25519`
6. 将生成的 SSH 公钥(id_rsa.pub)粘贴到GitHub上
7. 测试能否连接：`ssh -T git@github.com` 或者测试通过 HTTPS 端口的 SSH 是否可行 `ssh -vT -p 443 git@ssh.github.com`

设置本地Git使用 SSH 时的连接到远程的端口等配置：
+ MAC:
  1. 新建config文件配置远程连接端口： touch ~/.ssh/config
  2. 添加如下内容 
	```
	Host gjywgitlab.demo.net
	Port 223
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/id_rsa
	```
+ Windows:
1. 在用户的 .ssh 目录内建立 config 文件内容如上

### Git 配置
在Git中用户名和密码是保存在 git 配置文件中的。这个文件有全局配置和本地配置，全局配置指的是Git所有仓库都适用的配置，一般是 `~/.gitconfig` 文件中，本地配置指的是特定仓库适用的配置， 一般在 `.git/config` 文件中 。 `--global credential.helper store` 命令告诉Git将用户名和密码存储在本地 `.git-credentials` 文件中。

#### 查看/修改配置
```
git config --list //查看配置
git config --local --edit // 编辑本地配置文件
git config --global --edit // 编辑全局配置文件

git config --global http.sslVerify false // 修改全局配置
git config --local http.sslVerify false // 修改本地配置

git -c http.sslVerify=false clone https://example.com/path/to/git //执行命令时传递参数,跳过证书验证

//配置代理
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

#### 配置账号密码避免每次都要输入账号密码
1. `git config --global url."https://wangzhongzhu026484:wzz900119!@gjywgitlab.demo.net".insteadOf "http://gjywgitlab.demo.net"`
2. 配置用户名、密码：
   ```
   git config --global user.name 用户名
   git config --global user.password 密码
   git config --global user.email "1548429568@qq.com"
   ```
3. `git remote set-url origin https://{username}:{password}@github.com/{username}/project.git`

### git 常用命令
<center><img src="pics/git-frame.jpg" width="30%" /></center>

1. `git status [-s]` : 查看 git 状态，-s 代表 --short
2. `git diff`: 比较的是工作目录中当前文件和暂存区域快照之间的差异。 也就是修改之后还没有暂存起来的变化内容。
3. `git diff --staged`: 比对已暂存文件与最后一次提交的文件差异。
4. `git clean`: 删除工作区中未跟踪（不在版本库也未索引的）的文件
5. `git add`: 这是个多功能命令，可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态等。将这个命令理解为“精确地将内容添加到下一次提交中”而不是“将一个文件添加到项目中”要更加合适精确地将内容添加到下一次提交中
6. `git ls-files --satge`:查看暂存区的文件
7. `git diff` : 只显示尚未暂存的改动，而不是自上次提交以来所做的所有改动
8. `git diff --staged `:比对已暂存文件与最后一次提交的文件差异
9. `git rm filename` : 删除某个文件，提交后会同时删除版本库/暂存区和工作区的文件
10. `git rm --cached filename`:从暂存区和版本库中删除某个文件，提交后，暂存区/版本库中会删除这个文件，但是工作区磁盘上会保留
11. `git checkout head filename`:从暂存区恢复工作区中的文件，通常是想放弃对工作区中的修改或者误删了工作区文件
12. `git checkout <hash> <filename>`：恢复文件到某个提交状态
13. `git reset [HEAD|<h></h>ash] filename`:从版本库中恢复某个文件，当你删除了工作区中的文件，又删除了暂存区中的文件，想恢复文件可以用这个命令
14. `git commit`:将暂存区中的文件提交到版本库
15. `git commit --amend`：将本次提交与前一次的提交合并为一次提交
16. `git commit --amend --reset-author`; 修改提交的作者为 git config的作者
17. `git checkout branchname`:切换分支
18. `git checkout -b branchname`:切换分支,如果分支名不存在则基于当前分支新建一个分支
19. `git reset filename`:从版本库中还原某个文件到暂存区，当想撤销某次git add filename操作时使用，不加filename时，重置整个暂存区，工作区内容不受影响
20. `git reset`:恢复暂存区文件为版本库状态,工作区文件不会改变
21. `git reset --hard head~1`:用上一次提交的版本库状态恢复暂存区和工作区
22. `git log`: 查看提交历史记录
23. `git log -p -2`: 查看最近的2次提交及更改的内容
24. `git log filename`: 查看某个文件的历史提交记录
25. `git show <commit_id>`: 查看某次提交的信息信息
26. `git reflog`:查看引用历史记录
27. `git reset ref`:将版本库恢复到某次提交状态，适用于往前reset后又想往后reset
28. `git stash`:将当前工作区中的修改保存起来,并用版本库中的内容恢复暂存区和工作区，当开发到一半时，需要处理另一个bug时使用
29. `git stash pop`:将保存的内容恢复到工作区

### git 分支
0. `git checkout -b 本地分支名x origin/远程分支名x `: 在本地新建分支x，并自动切换到该本地分支x
1. `git branch -a`:查看所有分支
2. `git branch --merged|--no-merged`:查看已经合并到/尚未合并当前分支的分支
3. `git branch <BranchName>`:创建 BranchName 分支
4. `git checkout <BranchName>`:切换到 BranchName 分支
5. `git checkout -b <BranchName>`:在当前分支基础上新建 BranchName 分支，并切换到 BranchName 分支
6. `git brnch -d <BranchName>`:删除 BranchName 分支
7. `git merge <BranchName>`: 将 BranchName 合并到当前分支
8. `git merge --abort`: 中断合并
9. `git rebase <targetBranch> <sourceBranch>`: 它的原理是首先找到这两个分支(即源分支 sourceBranch 、变基操作的目标基底分支 targetBranch) 的最近共同祖先 C2，然后对比当前分支相对于该祖先的历次提交，提取相应的修改并存为临时文件，然后将当前分支指向目标基底 C3, 最后以此将之前另存为临时文件的修改依序应用。
10. `git checkout develop_2.0_backup -- docker-compose-builder.yaml docker-compose.yaml`：将一个分支的文件拷贝到当前分支

### git远程命令
1. `git clone`: 从远程库克隆
2. `git remote -vv` :显示远程主机
3. `git remote show <仓库名>` :显示远程主机详细信息
4. `git remote add <仓库名> <仓库地址>`:添加远程主机
5. `git remote rm <仓库名>`：删除远程主机
6. `git remote rename <原仓库名> <新仓库名>`：远程主机的改名
7. `git remote set-url <仓库名> <新的repo-url>`：修改远程仓库的地址
8. `git fetch <仓库名>`：访问远程仓库，从中拉取所有你本地仓库还没有的数据。 执行完成后，你将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看
9. `git fetch origin 远程分支名x:本地分支名x` : 在本地新建分支x，但是不会自动切换到该本地分支x，需要手动checkout
10. `git fetch <仓库名> <分支名>`：取回远程主机指定分支的更新
11. `git pull <仓库名> <远程分支名>:<本地分支名>`：取回远程主机某个分支的更新，再与本地的指定分支合并
12. `git pull <仓库名> <远程分支名>`：取回远程主机某个分支的更新，再与本地的当前分支合并
13. `git branch --set-upstream <本地分支名> <仓库名>/<远程分支名>` ：手动建立追踪关系
14. `git pull origin` : 如果当前分支与远程分支存在追踪关系，git pull就可以省略远程分支名
15. `git pull` :如果当前分支只有一个追踪分支，连远程主机名都可以省略
16. `git push remote localbranch:remotebranch` : 将本地localbranch分支推送到remote的remotebranch分支上
17. `git push remote localbranch` : 将本地localbranch分支推送到remote的同名分支上
18. `git push remote :remotebranch` :删除remote的remotebranch分支
19. `git push remote --delete remotebranch` :删除remote的remotebranch分支
20. `git push` : 如果当前分支只有一个追踪分支，那么主机名都可以省略
21. `git ls-remote -h -- ssh://git@gjywgitlab.demo.net:223/demo-app-platform/user-center-g/user-center-service.git`


#### 常见问题
1. windows乱码：  
    使用git bush时，使用git add XX 添加文件后，git status 发现中文文件名是数字形式，比如"\123\456\789.txt"，点击也无法打开，二使用ls，git log都可以显示中文，最后修改配置:
    `git config --global core.quotepath false ` 解决

2.  unable to access 'https://github.com/ScoopInstaller/Scoop/': SSL certificate problem: unable to get local issuer certificate
	`git config --global http.sslVerify false`

3. nable to access 'https://github.com/ScoopInstaller/Scoop/': schannel: next InitializeSecurityContext failed: Unknown error (0x80092012)
	`git config --system http.sslbackend openssl`

4. ssh: connect to host github.com port 22: Connection timed out
	```
	Host github.com
	Hostname ssh.github.com
	Port 443
	```

### 恢复
```
git reflog --date=iso

b4f3446 (HEAD -> develop_h5, wzz/develop_h5, new-branch-for-feature) HEAD@{2024-11-08 16:22:46 +0800}: checkout: moving from new-branch-for-feature to develop_h5
b4f3446 (HEAD -> develop_h5, wzz/develop_h5, new-branch-for-feature) HEAD@{2024-11-08 16:21:12 +0800}: checkout: moving from develop_h5 to new-branch-for-feature

<commit hash> HEAD@{7}: checkout: moving from main to new-feature
```
`commit hash` 就是你执行这个操作后所处的 commitId, 所以找到要恢复的执行操作的位置，然后记下 `commit hash`，然后：

```
git checkout -b new-branch-for-feature <commit hash>
git branch -f new-feature <commit hash>
```

## 删除某个历史commit
Once you push to the repo, you really don't want to go about changing history. However, if you are absolutely sure that nobody has pulled/fetched from the repo since your offending commit, you have 2 options.

If you want to remove the "bad" commit altogether (and every commit that came after that), do a `git reset --hard ABC` (assuming ABC is the hash of the "bad" commit's elder sibling — the one you want to see as the new head commit of that branch). Then do a git push --force (or `git push -f`).

If you just want to edit that commit, and preserve the commits that came after it, do a `git rebase -i ABC~`. This will launch your editor, showing the list of your commits, starting with the offending one. Change the flag from "pick" to "e", save the file and close the editor. 
In the editor that opens, find the line with the commit you want to delete, `change pick to drop, or simply delete the line`.
Then make the necessary changes to the files, and do a git commit -a --amend, then do `git rebase --continue`. Follow it all up with a `git push -f`.

I want to repeat, these options are only available to you if nobody has done a pull or fetch that contains your offending commit. If they have, doing these steps will just make matters worse.

## 提交一个change 到历史commit 中
1. Use `git stash` to store the changes you want to add.
2. Use `git rebase -i HEAD~10` (or however many commits back you want to see).
3. Mark the commit in question (a0865...) for edit by changing the word pick at the start of the line into edit. Don't delete the other lines as that would delete the commits.
4. Save the rebase file, and git will drop back to the shell and wait for you to fix that commit.
5. Pop the stash by using `git stash pop`.
6. Add your file with `git add <file>`.
7. Amend the commit with `git commit --amend --no-edit`.
8. Do a `git rebase --continue` which will rewrite the rest of your commits against the new one.
9. Repeat from step 2 onwards if you have marked more than one commit for edit.
10. If you have previously pushed the modified commits anywhere else, then you will have to push --force again to update them on the remote. However, the usual warnings about using --force apply, and you can easily lose other people's work if you are not careful and coordinate with them beforehand.

实战：
1. `git stash` -> 暂存需要提交到历史 commit 的更改
2. `git log` -> 查找需要提交到哪个commit，记下那个commit
3. `git rebase -i 99d14f7a77a6e8f91c40756aab76eded31f6151f` -> 会打开一个vim 编辑框, 将你想要提交到的 commit 行的 `pick` 改成 `edit`:
   <img src="pics/git-rebasei.png" width="50%" />
4. 保存文件并退出 vim 编辑
5. `git stash pop` -> 弹出提交的内容
6. `git add ./` -> 暂存要提交的内容
7. `git commit --amend` -> 提交内容
8. `git rebase --continue` -> 继续 git rebase 完成操作


## git 仓库过大清理
要对 Git 仓库进行清理和优化，回收无用数据，压缩对象，减小仓库体积，提高性能。可以使用下述命令：
```
git gc
git gc --prune=now  //会立即删除所有无法访问的对象，请慎用。
```
查看优化前后的效果：
```
du -sh .git
git count-objects -v
```

当 git 仓库太大，要推送的东西过多时，推送时可能会报错： `error: RPC failed; HTTP 500 curl 22 The requested URL returned error: 500`, 可以使用上述命令优化本地仓库大小。如果还有问题，可以尝试：
```
git config http.postBuffer 157286400
```

`git fsck` 是 Git 提供的一个用于检查仓库完整性和一致性的命令，全称是 “file system check”，其作用类似于文件系统中的 fsck 工具。 检测 Git 仓库中潜在的问题，包括：
| 问题类型   | 描述                             |
|------------|----------------------------------|
| dangling   | 悬空对象（未被任何引用）        |
| missing    | 丢失的对象或引用                |
| corrupted  | 损坏的对象（SHA 不匹配、解析失败） |
| invalid    | 非法对象结构                    |