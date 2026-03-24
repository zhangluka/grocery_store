---
name: 目录同步 shell 函数
overview: 在 ~/.zshrc 中添加一个 shell 函数 `syncdir`，使用 rsync 将源目录的所有文件（含隐藏文件）同步到目标目录，只增改不删除。
todos:
  - id: add-syncdir
    content: 在 ~/.zshrc 末尾追加 syncdir 函数
    status: completed
isProject: false
---

# 目录同步 shell 函数

## 方案

在 `[.zshrc](/Users/bobby/.zshrc)` 末尾追加一个 `syncdir` 函数，核心命令为：

```bash
rsync -av "$1/" "$2/"
```

关键点：

- `-a`：归档模式，递归复制，保留权限/时间戳/符号链接等，且默认包含隐藏文件
- `-v`：显示同步过程的详细输出
- 源路径末尾加 `/` 确保复制的是 A 目录**内容**而非 A 目录本身
- 不加 `--delete`，所以 B 中多出的文件不会被删除（只增改）

## 函数内容

```bash
syncdir() {
  if [[ $# -ne 2 ]]; then
    echo "用法: syncdir <源目录> <目标目录>"
    return 1
  fi
  if [[ ! -d "$1" ]]; then
    echo "错误: 源目录 '$1' 不存在"
    return 1
  fi
  mkdir -p "$2"
  rsync -av \
    --exclude='.git/' \
    --exclude='.gitignore' \
    --exclude='.gitattributes' \
    --exclude='.gitmodules' \
    "$1/" "$2/"
}
```

## 修改文件

- `[/Users/bobby/.zshrc](/Users/bobby/.zshrc)` - 在文件末尾追加上述函数

## 使用方式

```bash
syncdir /project/A /project/B
```

执行后会显示同步了哪些文件，B 目录不存在时会自动创建。
不会复制 .git/、.gitignore、.gitattributes、.gitmodules
