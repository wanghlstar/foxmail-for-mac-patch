# 故障原理完整分析
## 环境
- 客户端：Foxmail for macOS
- 协议：IMAP / SMTP
- 服务端：Dovecot IMAP
- 现象：IMAP登录成功，SMTP发信正常；收取邮件提示无新邮件；Windows版Foxmail同账号正常。

## 问题A：AUTHENTICATE PLAIN 参数传递BUG
### 现象
Mac Foxmail在PLAIN认证分支，组装IMAP命令时将凭据参数写死为NULL，输出报文：
`A AUTHENTICATE PLAIN (null)`
Dovecot收到非法参数直接拒绝本次认证流程，收信流程中断。

### 分支逻辑
程序存在三条认证分支：
1. XOAUTH2：QQ邮箱OAuth令牌登录，逻辑正常，本次不做任何改动。
2. LOGIN密码分支：存在缺陷。
3. PLAIN密码分支：存在缺陷。

### 修复思路
修改LOGIN、PLAIN两条分支的指令，绕过损坏的AUTHENTICATE PLAIN调用，改用标准IMAP `LOGIN user password` 交互，行为对齐Thunderbird、雁信客户端。

## 问题B：IMAP响应结束判定硬编码字符串匹配
### 现象
客户端不使用IMAP标准tag机制判断命令结束，而是用固定字符串字面量匹配服务端返回行尾部。
Dovecot默认在OK tagged响应末尾追加耗时注释：
`A OK [READ-WRITE] Select completed (0.001 + 0.000 secs).`
后缀 `(xxx + xxx secs).` 不在客户端内置匹配列表内。客户端判定命令永远没有结束，等待60秒超时，文件夹同步失败，无法检测新邮件。

### 修复思路
修改程序内硬编码字面量表，替换4条字符串：新增支持 ` secs).` 和 `Logged in` 匹配，兼容Dovecot带计时后缀的返回报文。

## 补充边界说明
1. 本次修改为原地字符串/指令补丁，不新增代码段，最小改动。
2. 补丁仅适配带计时后缀的Dovecot IMAP服务端；其他IMAP服务器无该后缀不受影响。
3. Foxmail版本升级会导致Mach-O文件布局变化，偏移地址全部失效。
