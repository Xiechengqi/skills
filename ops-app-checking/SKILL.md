---
name: ops-app-checking
description: 使用基本的运维手段检查当前环境中的应用状态
license: 完整条款见 LICENSE.txt
---

# 应用状态分析

使用基本的运维手段检查应用状态

## 3、通过系统进程工具检查应用状态
无法匹配则代表进程不在运行，提示用户
```bash
ps -ef | grep -v grep | grep -i [应用名] && echo "[应用名] 正在运行 ..." || echo "[应用名] 不在运行 ..."
```

## 2、通过应用管理工具查看状态
通过上一步进程检查结果，尝试以下应用管理工具。如果都没有匹配，询问应用进程管理命令
```bash
# systemctl
systemctl status [应用名] --no-pager && echo "[应用名] 使用 systemctl 管理 ..."

# docker
docker ps | grep [应用名] && echo "[应用名] 使用 docker 管理 ..."
```

## 3、调用 gemini 分析应该日志是否有问题
1. 先将最新 1000 行日志写入临时日志文件 /tmp/ask-gemini.txt
```bash
# 应用日志文件
tail -1000 [应用日志文件] > /tmp/ask-gemini.txt

# docker
docker logs --tail 1000 [应用名] > /tmp/ask-gemini.txt

# systemctl
systemctl status [应用名] --no-pager > /tmp/ask-gemini.txt
```
2. 调用 gemini 分析日志
```
echo '======= 找出上面日志中的异常日志，分析问题根因并分类归纳输出(完全基于文本分析，不调用外部工具) ========' >> /tmp/ask-gemini.txt
cat /tmp/ask-gemini.txt | gemini
```
