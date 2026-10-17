# 手工补丁 + ad-hoc重签名操作指南
> 重要前提
> 1. 完全退出Foxmail，关闭所有进程。
> 2. 准备原版Foxmail，先手动备份Mach-O文件，记录sha256哈希。
> 3. 确认Foxmail版本与文档偏移匹配，版本不一致不要操作。
> 4. 需要macOS命令行工具，python3、codesign。

## 步骤1：定位目标二进制
目标路径：
`/Applications/Foxmail.app/Contents/MacOS/Foxmail`

## 步骤2：读取对应偏移区段，校验原始字节
使用python读取指定范围字节，和补丁信息表内【可接受原始字节】对比。
任意区段不匹配 → 中止，版本不兼容。

## 步骤3：原地写入目标字节
按补丁表，对每个区段替换为目标十六进制字节。
写入完成后，重新读取全部9处区段做回读校验，确认写入成功。

## 步骤4：编写entitlements配置plist
创建临时plist文件，内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.security.app-sandbox</key>
	<true/>
	<key>com.apple.security.cs.disable-library-validation</key>
	<true/>
	<key>com.apple.security.files.bookmarks.app-scope</key>
	<true/>
	<key>com.apple.security.files.user-selected.read-write</key>
	<true/>
	<key>com.apple.security.network.client</key>
	<true/>
	<key>com.apple.security.network.server</key>
	<true/>
	<key>com.apple.security.personal-information.addressbook</key>
	<true/>
	<key>com.apple.security.personal-information.calendars</key>
	<true/>
	<key>com.apple.security.print</key>
	<true/>
</dict>
</plist>
