# 执行adhoc签名
codesign --force --sign - --entitlements /tmp/foxmail-entitlements.plist /Applications/Foxmail.app

# 签名校验
codesign --verify --verbose=1 /Applications/Foxmail.app
codesign -dv /Applications/Foxmail.app
