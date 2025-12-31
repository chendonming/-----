# git 统一更改换行符

## 创建 .gitattributes 文件

根目录下创建 `.gitattributes`文件

```
# 强制所有文本文件在检入仓库时使用 LF，且在检出时保持 LF
* text=auto eol=lf
```

## 刷新项目

设置好上述文件后，运行以下命令让 Git 重新规格化现有文件：

```bash
# 1. 保存当前工作区（确保没有未提交的修改）
git add . -u
git commit -m "Save current work"

# 2. 删除 Git 索引并强制刷新
git rm --cached -r .
git reset --hard
```

> 注意：这会重置你本地未提交的修改，请务必先 commit 或 stash。