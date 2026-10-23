# 附带问题：Mac 版「已删除」失效（服务器缺废纸篓邮箱）

> 与二进制补丁**相互独立**：补丁只改认证分支与响应完成判定，不碰任何文件夹映射。
> 打补丁之前此问题同样存在，只是被「每条命令干等 60 秒超时」掩盖了。

## 现象
- Foxmail 里删除邮件，邮件不动，日志三连：
  ```
  INBOX move mail to Deleted Messages
  A NO Mailbox doesn't exist: Deleted Messages (0.001 + 0.000 secs).
  Copy mail error! Src folder: INBOX Des folder: Deleted Messages
  ```
- 同一账号在 Windows 版 Foxmail 删除正常。

## 根因
1. Mac 版 Foxmail 的「删除」= IMAP `COPY` 到废纸篓邮箱 + 源箱打删除标记。
2. 它把系统文件夹的服务器名**硬编码成英文**（1.5.9 build 14685，二进制约 `0x114fcdf` 处的候选表）：
   `drafts/Drafts`、`sent/Sent Messages`、`trash/Deleted Messages`、`junk/Junk`、`spam`、
   `Flagged`、`All`、`Important`、`INBOX`、`Draft`、`Trash`。
3. 账号建成后，「已删除」的 serverpath 即被固定写成 `Deleted Messages`（本地 `boxes` 表可直接查）。
   服务器上没有这个邮箱 → COPY 必然失败。
4. Windows 版不受影响，是因为该账号在 Windows 端被识别为「腾讯企业邮」专用类型，走另一套实现，
   不依赖这份硬编码表；Mac 端按通用 IMAP 添加（`protocol=1`），才会踩到。

## 为什么与补丁无关
- 失败是服务器明确返回 `NO Mailbox doesn't exist`——邮箱不存在，与认证、响应解析无关。
- 恰恰因为补丁命中了 ` secs).` 后缀，这些 NO 回包才能被立刻解析、才会在日志里现形。
- 日志中 `Drafts`/`Sent`/`Junk` 的失败清一色是 `curl Error-Result:56`（对端断开），
  属于初始同步时连接被重置；**从未**出现 `Mailbox doesn't exist: Drafts`，
  说明这些邮箱服务器上是有的 → 不要因此再去建 `Drafts`，否则会多出一个草稿箱。

## 修复：在服务器补建废纸篓邮箱
「已删除」的 serverpath 本来就指向 `Deleted Messages`，建好即通，无需改任何客户端配置：

```sh
./create-deleted-mailbox.sh imap.example.com user@example.com
```

验证：日志出现 `<账号> Deleted Messages 同步邮件状态成功，个数：1`、
`枚举邮件完成 文件夹邮件个数：1`，且此后再无 `Copy mail error`。

其他可选方案：
- 查网页邮箱里「已删除」的真实名字，改本地 `boxes` 表对应行的 serverpath（账号刷新可能被覆盖）。
- 添加账号时选「腾讯企业邮」专用类型，走 Windows 端那套实现。

## 坑：在 Foxmail 界面里「新建文件夹」不一定管用
实测 Foxmail 确实发出了 `ImapCommandCreate` 且 34ms 就记为成功，但随后 COPY 仍报邮箱不存在——
那条连接正是随后被服务器断开的那条（同一现象的 curl 56），命令实际没落地。
用独立连接创建（下面的脚本）可验证、可重试。

## 附：补建脚本
脚本只执行 `LIST` / `CREATE` / `SELECT`，不读取、不修改任何邮件；
密码用 `getpass` 读入，不回显、不进命令行参数、不落盘；幂等，邮箱已存在则跳过创建。
默认只补建 `Deleted Messages`；确认服务器真没有 `Drafts` 才加 `--drafts`。

```sh
#!/bin/sh
# ============================================================
# 修复 Foxmail for Mac「已删除」失效：在服务器补建废纸篓邮箱
# 背景与原理见同目录 03-deleted-mailbox-fix.md
# ------------------------------------------------------------
# 用法：
#   ./create-deleted-mailbox.sh <IMAP主机> <账号> [--drafts]
# 例：
#   ./create-deleted-mailbox.sh imap.example.com user@example.com
#   ./create-deleted-mailbox.sh imap.example.com user@example.com --drafts
#
# 约定：
#   - 密码走 getpass，不回显、不进命令行参数、不落盘。
#   - 只执行 LIST / CREATE / SELECT，不碰任何邮件。
#   - 幂等：邮箱已存在则跳过创建。
#   - 默认只补建 "Deleted Messages"；确认服务器真没有 Drafts 才加 --drafts。
# ============================================================
set -e

[ $# -ge 2 ] || { echo "用法: $0 <IMAP主机> <账号> [--drafts]"; exit 1; }
HOST=$1
ACCT=$2
if [ "$3" = "--drafts" ]; then DRAFTS=1; else DRAFTS=""; fi

python3 - "$HOST" "$ACCT" "$DRAFTS" <<'PY'
import imaplib, getpass, sys

host, acct, want_drafts = sys.argv[1], sys.argv[2], sys.argv[3] == "1"
try:
    pw = getpass.getpass("IMAP 密码: ")
except EOFError:
    sys.exit("读不到密码：本脚本需要在真实终端里交互运行（不能管道/重定向输入）。")

# Foxmail 硬编码的系统文件夹服务器名，见 03-deleted-mailbox-fix.md
FOLDERS = ["Deleted Messages"] + (["Drafts"] if want_drafts else [])

M = None
for port, ssl in ((993, True), (143, False)):
    try:
        M = imaplib.IMAP4_SSL(host, port) if ssl else imaplib.IMAP4(host, port)
        M.login(acct, pw)
        print("已连接 %s:%d（%s）" % (host, port, "SSL" if ssl else "明文"))
        break
    except Exception as e:
        print("  %s:%d 登录失败：%s" % (host, port, e))
        M = None
if M is None:
    sys.exit("两个端口都无法登录，请检查主机/账号/密码。")

print("== 当前文件夹 ==")
for line in M.list()[1]:
    print("   ", line.decode(errors="replace").strip())

for name in FOLDERS:
    typ, data = M.create('"%s"' % name)
    detail = data[0].decode(errors="replace") if data and data[0] else ""
    print("CREATE %-18s -> %s %s" % (name, typ, detail))
    typ, data = M.select(name)
    detail = data[0].decode(errors="replace") if data and data[0] else ""
    print("SELECT %-18s -> %s %s" % (name, typ, detail))

print("== 创建后 ==")
for line in M.list()[1]:
    print("   ", line.decode(errors="replace").strip())
M.logout()
print("完成：回 Foxmail 删一封邮件验证；左侧没刷新就重启 Foxmail。")
PY
```
