---
title: git避免提交错乱
date: 2021-09-01 15:33:26
tags:
- 生产小经验
- git
---



# 命令

```bash
git config --local pull.rebase true 
```

# 解决冲突

base, local, remote 生成三个文件，只读。在3个文件中，选择你想使用的行，一般情况：

> 1. 家里开发忘记推了
> 2. 公司开发，继续开发后续的，推了
> 3. 家里回来拉就冲突，家里是旧的，由于基点是base文件，然后对比冲突。只需要把公司那部分新的隔离，家里这部分隔离开就可以了

先配置becompare 为解决冲突的工具, 使得调用 `git mergetool`就自动使用这个工具完成解决。

1. 获取bcompare执行路径

   ```bash
   D:\Program Files (x86)\BCompare\BCompare.exe
   ```

2. 配置git mergetool工具

   ```bash
   git config --global merge.tool bc4
   git config --global mergetool.bc4.cmd "\"D:\Program Files (x86)\BCompare\BCompare.exe\" \"\$LOCAL\" \"\$REMOTE\" \"\$BASE\" \"\$MERGED\""
   git config --global mergetool.bc4.trustExitCode true
   git config --global mergetool.keepBackup false
   ```

   > 第2行就是配置这个bcompare路径

3. 当我们`git pull`之后出现冲突，就执行

   ```bash
   git mergetool
   ```

4. 出现bcompare 界面，并有4个窗口

   上面3个只读，下面1个是冲突的文件。在冲突的文件上，左侧找红色的部分，然后在上面决定使用左侧还是右侧。

   为了理清3个文件的关系，可以使用查看3个文件的工具，比如我写的markdown就使用typora，依次查看BASE, LOCAL, REMOTE看看3个文件写的内容哪里不同，再结合bcompare完成配置，使用哪个版本，或者直接隔离大量的行，当成一个整体使用。

   

