
``` bash
# 复制微信应用，创建分身副本
sudo cp -R /Applications/WeChat.app /Applications/wx.app
# 修改分身标识，避免系统冲突
# 通过修改应用的唯一标识符（Bundle ID），让系统认为「wx」是独立应用，从而绕过微信的进程互斥限制
sudo /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier com.tencent.xinWeChat.dual" /Applications/wx.app/Contents/Info.plist
# 重新签名应用，解决「损坏」提示
# 修改标识后系统会认为应用未签名，直接打开会提示「已损坏」
sudo codesign --force --deep --sign - /Applications/wx.app
```
