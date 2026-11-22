---
name: pennywise
description: Use when recording expenses or budgets via pennywise CLI.
version: 2.0.0
license: MIT
metadata:
  hermes:
    tags: [pennywise, cli, expense, accounting, npm]
    related_skills: [email/himalaya]
---

## When to Use

- 主人说「记一笔 X 元 …」→ 直接跑 `pennywise add`，不要问、不要先跑 ingest 以外的动作
- 主人说「拉账单/同步账单」→ 跑 `scripts/auto-sync.sh`，把结果一行摘要回给主人
- 主人说「这个月花了多少 / 查一下 X」→ summary/list 直接答；**非必要不主动翻账本**
- 提到 expense/记账/预算/summary 且上下文是个人财务 → 本 skill

# PennyWise（coder 视角）

## 仓库归属（别搞混）

- **主人的完整项目**：`github.com/hehehe1234567894/PennyWise` —— Web/PWA + Node 服务端 + SQLite，
  版本日期制 v26.x，他本人维护。要改它：fork 到 MIEOEMI → 开分支 → PR，**不要直接推、也不要另起炉灶**
- **本 CLI**：`github.com/MIEOEMI/pennywise-cli`（原 MIEOEMI/pennywise，2026-11-22 改名）——
  coder 从零写的零依赖命令行版，和上面那个是两条线，互不同步

## 分工铁律（主人 2026-11-16 定）

- **管道全自动**：邮箱拉取(fetch-bills.py)→导入(ingest)由 cron 机器跑，**不需要 coder 每次手动执行**
- **coder 只负责审核和查阅**：一般情况下不审核、非必要不查阅
- 账单文件投递进 inbox 后由机器合并进账本，**不是 coder 手动 merge**
- ingest 失败 / failed/ 目录有文件 / 主人点名要报告时，coder 才介入看

## 自动管道（已配置 cron）

```
crontab（agent 用户）每小时 :40 → /bin/bash scripts/auto-sync.sh
  ├─ fetch-bills.py   163 邮箱拉账单邮件（认证前 ID 绕 Unsafe Login）→ ~/.pennywise/inbox/
  └─ pennywise ingest  inbox → data.json（去重；成功文件归档 processed/，坏文件 failed/）
日志：~/.pennywise/auto-sync.log
```

脚本自己 export `PATH=$HOME/.local/bin:…` 和 `TZ=Asia/Shanghai`——cron 的默认 PATH 里没有
`~/.local/bin`（pennywise 装在那里），少这一行会 `pennywise: command not found`。
2026-11-19 之前这条 cron 并不存在（只有手动跑过），现已装好并用「每分钟跑一次」实测通过后改回 :40。

手动触发同一条链：`bash scripts/auto-sync.sh`，然后 `tail -5 ~/.pennywise/auto-sync.log` 看结果。

## 命令速查

| 命令 | 说明 |
|------|------|
| `pennywise add 32.5 -n 午饭 -c 餐饮 -s 美团` | 记支出（-s 商家，-d 日期默认今天） |
| `pennywise income 3000 -n 生活费` | 记收入 |
| `pennywise show <ID前缀>` | 单条详情 |
| `pennywise list [-q 关键词] [--cat] [--type] [--from/--to] [-m 月]` | 筛选查询（-q 搜商家/备注/分类） |
| `pennywise summary [月]` | 月汇总+分类占比+预算 |
| `pennywise budget 2000 / budget clear` | 月度预算 |
| `pennywise export --format csv/json --out 文件 [--no-note] [--mask] [--replace 旧=新]` | 导出（可剔除/打码/替换） |

## 行为要点

- **账户互转不算收支**：`category=账户存取`（小荷包/余额宝 转入转出）导入时打 `transfer: true`，
  `summary` 单列「账户互转 N 笔 · 转入/转出」不计收入支出与预算；`list --type 互转` 可查。
  给主人报月度数时直接念 summary 的收入/支出，**不要**把互转加回去
- **「不计收支」不等于不花钱**：支付宝把真实消费（食堂/交通/日用）也标成不计收支，只有分类属于
  `账户存取/退款/转账/转账红包/信用借还` 才跳过，其余按支出入账；「交易关闭」一律跳过
- **去重先看订单号**：有 `ref`（支付宝/微信订单号）按 ref 比，没有才退回「日期+类型+金额+分类+备注+商家」
- 时区坑：本机 /etc/localtime=UTC，cron 进程北京时间 00:00-08:00 记账会落到前一天 →
  替主人记账一律显式传 `-d`（当天按 Asia/Shanghai 算）
- 测试用 `PENNYWISE_HOME=/tmp/xxx pennywise …`，否则写进真实账本；注意 terminal 会话里 export 会一直生效，
  测完记得 `unset PENNYWISE_HOME` 再操作真实账本
- 授权码只在 `~/.config/himalaya/config.toml`（600），fetch-bills.py 从那里读，不落第二份
- 安装：`npm install -g github:MIEOEMI/pennywise`（npmjs 公共源不发，主人已定）；
  本地装的是 `~/.local/lib/node_modules/pennywise -> ~/workspace/pennywise` 的软链，改完源码立即生效
- 改完源码记得 `git commit` + `git push origin HEAD`（MIEOEMI 账号），别留一堆未提交的改动
