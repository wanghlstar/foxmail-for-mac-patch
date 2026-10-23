# Foxmail for macOS IMAP 兼容性问题研究
本项目仅用于个人技术研究与IMAP协议互操作性分析，**不提供可执行补丁脚本、不分发任何Foxmail二进制程序**。

## 现象
- IMAP账号登录成功，SMTP发送邮件正常。
- 点击收取邮件，客户端提示无新邮件，服务器实际存在邮件。
- 同IMAP账号在Windows版Foxmail工作正常；Dovecot服务端出现该故障。

## 根因简述
1. **认证分支BUG**
Mac版Foxmail在`AUTHENTICATE PLAIN`密码认证分支将凭据参数硬编码为NULL，发送非法命令 `A AUTHENTICATE PLAIN (null)`，被Dovecot拒绝。
修复思路：将LOGIN/PLAIN两条认证分支修改，强制使用标准IMAP LOGIN指令；XOAUTH2令牌分支完全保留不作改动。

2. **IMAP响应结束判定逻辑缺陷**
客户端依靠固定硬编码字符串字面量匹配，判断IMAP命令响应是否结束。
Dovecot返回的OK结果附带计时后缀：`(0.001 + 0.000 secs).`，无法命中内置匹配表。客户端认为命令永远未结束，等待60秒超时，文件夹同步失败。
修复思路：修改内置字面量表，增加匹配` secs).`和`Logged in`。

3. **系统文件夹名硬编码（附带问题，与上述补丁无关）**
Mac版把「已删除」的服务器名硬编码为`Deleted Messages`，删除邮件=IMAP COPY到该邮箱；服务器上没有同名邮箱时返回`NO Mailbox doesn't exist`，删除失效。Windows版因账号走「腾讯企业邮」专用类型而不受影响。修复：在服务器补建同名邮箱即可，见 `03-deleted-mailbox-fix.md`。

## 重要提示
1. 所有修改操作仅允许在**本人合法持有的Foxmail本地副本**上做个人学习研究。
2. 补丁偏移地址绑定**特定Foxmail Mac版本**，软件更新后偏移会失效，强行修改会损坏程序。
3. 修改Mach-O二进制后，必须执行ad-hoc重签名并配置对应entitlements。
4. Foxmail自动更新会覆盖所有二进制修改，更新后需要重新手工操作。

## 文档目录
- docs/01-problem-analysis.md 完整故障原理分析
- docs/02-patch-info.md 补丁点位、原始字节与目标字节对照表
- docs/03-manual-operation.md 手工补丁与重签名操作步骤
- docs/04-reverse-notes.md 逆向与抓包分析笔记
- docs/05-faq.md 常见问题与回滚方案
- 03-deleted-mailbox-fix.md 附带问题：已删除失效与废纸篓邮箱补建（含脚本）

## 配套脚本
- apply-foxmail-patch.sh 二进制补丁 + 重签名（优先用固定自签证书 `Mac Local Self Signed`，找不到时回退 ad-hoc）
- create-deleted-mailbox.sh 补建服务器废纸篓邮箱（修复删除失效）
  （注：`.gitignore` 忽略了 `*.sh`，脚本不入库，仅存于本地工作目录）

## 免责
使用本仓库全部资料前，请阅读根目录下 `DISCLAIMER.md`。所有操作风险由使用者自行承担。
