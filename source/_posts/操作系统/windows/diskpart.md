---
title: diskpart使用
tags:
- command
- windows
---

# 使用diskpart命令格式化硬盘

```bash
# 进入diskpart命令行
diskpart

# 如下命令都在diskpart命令行中
# 查看帮助
help
# 列出磁盘
list disk
# 选中磁盘
select disk <磁盘序号>
# 清除磁盘
clean
# 创建主分区
create partition primary
# fat32格式格式胡
format fs=fat32 quick
# 激活分区
active
# 分配盘号
assign
```

