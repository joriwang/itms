---
title: 基于 WebClip 技术的自定义应用图标
date: 2020-11-02 19:04:28
tags:
 - WebClip
 - Data 协议
 - base64
 - iOS 美化
categories:
 - iOS
---

最近在做 iOS 美化相关的工作。主要集中在 APP 图标美化、类主题的美化（壁纸+app美化）、锁屏动态效果（主屏不支持）。

这是美化相关文章的第一篇，记录了 WebClip 如何实现 APP 图标美化及其涉及的相关技术

# 原理

WebClip 本质是桌面 web 书签，通过 Safari 来打开相关的应用。因此既可以通过 deep link 方式启动三方应用，又可以通过 scheme 来启动三方应用。为了实现批量设置应用图标简化用户的使用流程，我们使用 `mobileconfig` 描述文件打包相关设置。

`mobileconfig` 是 Apple MDM 的一部分。是一种 plist 文件。因此生成或修改这种文件有两种方式：Apple Configurator 2 或者直接修改 plist 文件。

下面是一个 mobileconfig 文件的内容 demo

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
 <key>PayloadContent</key>
 <array>
   <dict>
     <key>FullScreen</key>
     <true/>
     <key>Icon</key>
     <data>icon base64 data</data>
     <key>IsRemovable</key>
     <true/>
     <key>Label</key>
     <string>ScreenDisplayName</string>
     <key>PayloadDescription</key>
     <string>add web clip</string>
     <key>PayloadDisplayName</key>
     <string>Web Clip (信息)</string>
     <key>PayloadIdentifier</key>
     <string>identifier value</string>
     <key>PayloadOrganization</key>
     <string>organization name</string>
     <key>PayloadType</key>
     <string>com.apple.webClip.managed</string>
     <key>PayloadUUID</key>
     <string>uuid</string>
     <key>PayloadVersion</key>
     <integer>1</integer>
     <key>Precomposed</key>
     <false/>
     <key>URL</key>
     <string>mqq://aa.bb</string>
   </dict>
</array>
<key>PayloadDescription</key>
<string>description for mobileconfig</string>
<key>PayloadDisplayName</key>
<string>displayname for mobileconfig</string>
<key>PayloadIdentifier</key>
<string>identifier value</string>
<key>PayloadOrganization</key>
<string>organization value</string>
<key>PayloadRemovalDisallowed</key>
<false/>
<key>PayloadType</key>
<string>Configuration</string>
<key>PayloadUUID</key>
<string>uuid</string>
<key>PayloadVersion</key>
<integer>1</integer>
</dict>
</plist>
```

其中最重要的是 `PayloadUUID`、`PayloadIdentifier`、`URL`、`Icon`、`Label`。相同 `PayloadUUID` 的 mobileconfig 不会重复安装。`URL` 用于指定用户点击之后要跳转的页面。`Icon` 用于指定在桌面上的显示图标。`Label` 用于指定在桌面上显示的名字。

通过下载和安装 mobileconfig，就可以将其中包含的 webclip 添加到桌面上。

# mobileconfig 的下载探索

为了实现用户方便下载描述文件，我们尝试了如下方式，并一一失败

**data 协议**

通过把下载页面编码成 `data:text/html;base64,base64data==` 的形式，通过 `UIApplication` 的 `open` api 来跳转 Safari 实现无服务器依赖的下载流程。

失败原因：该 api 不支持 data 协议，仅支持 http、https、facetime、tel、mailto 等少数协议

下面是一个 html 样例。

```
data:text/html;base64,PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KPGhlYWQ+CiAgPG1ldGEgY2hhcnNldD0iVVRGLTgiPgogIDxtZXRhIG5hbWU9InZpZXdwb3J0IiBjb250ZW50PSJ3aWR0aD1kZXZpY2Utd2lkdGgsIGluaXRpYWwtc2NhbGU9MS4wIj4KICA8dGl0bGU+aWNvbjwvdGl0bGU+CjwvaGVhZD4KPHNjcmlwdD4KICB2YXIgd2lkdGggPSB3aW5kb3cuc2NyZWVuLndpZHRoOwogIHdpZHRoID0gTWF0aC5taW4od2lkdGgsIDc1MCk7CiAgZG9jdW1lbnQuZG9jdW1lbnRFbGVtZW50LnN0eWxlLmZvbnRTaXplID0gd2lkdGggLyAzLjc1ICsgJ3B4JzsKPC9zY3JpcHQ+CjxzdHlsZT4KICBib2R5IHsKICAgIGJhY2tncm91bmQtY29sb3I6ICM3RDhFQTE7CiAgICB3aWR0aDogMTAwJTsKICAgIGhlaWdodDogMTAwJTsKICAgIG1hcmdpbjogMDsKICAgIHBhZGRpbmc6IDA7CiAgfQogIC50aXBzIGltZyB7CiAgICB3aWR0aDogMC44NHJlbTsKICAgIGhlaWdodDogMC43OXJlbTsKICAgIHBvc2l0aW9uOiBhYnNvbHV0ZTsKICAgIHRvcDogY2FsYyg1MCUgKyA1NXB4KTsKICAgIG1hcmdpbi1sZWZ0OiAyLjYwcmVtOwogIH0KICAudGlwcyBwIHsKICAgIGNvbG9yOiByZ2JhKDI1NSwgMjU1LCAyNTUsIDAuOSk7CiAgICBmb250LXNpemU6IDAuMThyZW07CiAgICBmb250LXdlaWdodDogNTAwOwogICAgdGV4dC1zaGFkb3c6IDBweCAwcHggMHB4IHJnYmEoNjUsIDczLCA4MywgMC41Nik7CiAgICB3aWR0aDogMTAwJTsKICAgIHRleHQtYWxpZ246IGNlbnRlcjsKICAgIG1hcmdpbjogMDsKICAgIHBvc2l0aW9uOiBhYnNvbHV0ZTsKICAgIHRvcDogY2FsYyg1MCUgKyA2MHB4ICsgMC43OXJlbSk7CiAgfQogIC50aXBzIHNwYW4gewogICAgY29sb3I6IHJnYmEoMjU1LCAyNTUsIDI1NSwgMC41KTsKICAgIGZvbnQtc2l6ZTogMC4xM3JlbTsKICAgIHdpZHRoOiAxMDAlOwogICAgZGlzcGxheTogYmxvY2s7CiAgICB0ZXh0LWFsaWduOiBjZW50ZXI7CiAgICBwb3NpdGlvbjogYWJzb2x1dGU7CiAgICB0b3A6IGNhbGMoNTAlICsgNzBweCArIDFyZW0pOwogIH0KICAubmV4dCB7CiAgICB3aWR0aDogMS43NnJlbTsKICAgIGhlaWdodDogMC43MnJlbTsKICAgIHBvc2l0aW9uOiBhYnNvbHV0ZTsKICAgIHRvcDogODQlOwogICAgbGVmdDogNTAlOwogICAgdHJhbnNmb3JtOiB0cmFuc2xhdGVYKC01MCUpOwogIH0KICAubmV4dCBpbWcgewogICAgd2lkdGg6IDEwMCU7CiAgICBoZWlnaHQ6IDEwMCU7CiAgICBwb3NpdGlvbjogYWJzb2x1dGU7CiAgICBsZWZ0OiA1MCU7CiAgICB0b3A6IDUwJTsKICAgIHRyYW5zZm9ybTogdHJhbnNsYXRlKC01MCUsIC01MCUpOwogIH0KPC9zdHlsZT4KPGJvZHk+CiAgPGRpdiBjbGFzcz0idGlwcyI+CiAgICA8aW1nIHNyYz0iaHR0cDovL2NhY2hlLm5leHQubW94aXUuY29tL2NvbW1vbi9iY2Q5ZDQ5MWMwM2I5MGVkNzBmNjlmM2Q3NTQwYjA2NGRjZjBlNzY3Ij4KICAgIDxwPuato+WcqOeUn+aIkOWbvuagh++8jOivt+eCueWHu+WFgeiuuDwvcD4KICAgIDxzcGFuIG9uY2xpY2s9IndpbmRvdy5sb2NhdGlvbi5yZWxvYWQoKSI+5aaC5p6c5LiN5bCP5b+D54K55LqG5b+955Wl5oyJ6ZKuLCDor7fngrnlh7vliLfmlrDpobXpnaLph43or5Xlk6Z+PC9zcGFuPgogIDwvZGl2PgogIDxhIGhyZWY9Imh0dHBzOi8vZWxmLnN0YXRpYy5tYWliYWFwcC5jb20vYXNzZXRzL2h0bWwveHlqLXBydWUvcGljZ3VpZGUuaHRtbCIgY2xhc3M9Im5leHQiPgogICAgPGltZyBzcmM9Imh0dHA6Ly9jYWNoZS5uZXh0Lm1veGl1LmNvbS9jb21tb24vZjUwYWVlOGMyNTY4MWNiZDhlMjFiODg5MmE2ZTFjZDFlNmQ3OTk1MCI+CiAgPC9hPgo8L2JvZHk+CjxzY3JpcHQ+CiAgdmFyIGMgPSAnRklMRUNPTlRFTlQnCiAgdmFyIGEgPSBkb2N1bWVudC5jcmVhdGVFbGVtZW50KCdhJykKICBhLmRvd25sb2FkID0gJ0ZJTEVOQU1FLm1vYmlsZWNvbmZpZycKICB2YXIgYiA9IG5ldyBCbG9iKFtjXSkKICBhLmhyZWYgPSBVUkwuY3JlYXRlT2JqZWN0VVJMKGIpCgogIGRvY3VtZW50LmJvZHkuYXBwZW5kQ2hpbGQoYSkKICBhLmNsaWNrKCkKICBkb2N1bWVudC5ib2R5LnJlbW92ZUNoaWxkKGEpCjwvc2NyaXB0Pgo8L2h0bWw+Cg==
```

**内置 web 服务器**

使用 `GCDWebServer` 在应用内架设 Web 服务，通过 `http://127.0.0.1:9000` 来实现 mobileconfig 的下载和使用引导。

失败原因：应用进入后台后，相关端口无法接收数据

# mobileconfig 的签名处理

mobileconfig 文件不签名也可以使用，但是系统会给用户不安全提示，并且比签名版多一个确认步骤。

在使用 Apple Configurator 2 生成描述文件时，可以选择本机的开发者证书签名该文件。也可以通过 openssl 用 ssl 证书签名该文件。

使用 openssl 签名的命令如下

```
openssl smime -sign -in webclip.mobileconfig -out webclip_signed.mobileconfig -signer signer.pem -inkey my.pem -certfile ca.pem -outform der -nodetach
```

* signer.pem 对应的是域名使用的证书（公钥证书）
* my.pem 对应的是域名证书的私钥
* ca.pem 对应的是证书链

## 关于证书链

如果你用的是 `Let's encrypt` ，证书链可以在 web 工具的配置文件中找到。如果你用的是商业 CA，请联系对方。

如果你想使用开发者证书来通过命令行签名，可以通过 https://www.apple.com/certificateauthority/ 下载相应的证书组成证书链

# 关于跳转页面

点击桌面上的 webclip 后，safari 会跳转到相应的链接。如果使用 scheme 方式跳转应用，点击后会先显示一个全屏的白色页面，然后打开对应的应用。为了美化这个白色页面可以使用 `data:` 协议制作一个中间页，在这个页面里通过 js 打开对应 app，这样，跳转流程会更美观。

下面是 *dop 主题图标* 这款应用使用到的中间页（侵删。[463398425@qq.com](mailto:463398425@qq.com)）

```
data:text/html;base64,PGh0bWwgbGFuZz0iemgiPjxoZWFkPjxtZXRhIGNoYXJzZXQ9IlVURi04Ij48bWV0YSBodHRwLWVxdWl2PSJjYWNoZS1jb250cm9sImNvbnRlbnQ9Im5vLWNhY2hlIi8+PG1ldGEgY29udGVudD0ieWVzIm5hbWU9ImFwcGxlLXRvdWNoLWZ1bGxzY3JlZW4iPjxtZXRhIGNvbnRlbnQ9InllcyJuYW1lPSJhcHBsZS1tb2JpbGUtd2ViLWFwcC1jYXBhYmxlIj48bWV0YSBjb250ZW50PSJ3aGl0ZSJuYW1lPSJhcHBsZS1tb2JpbGUtd2ViLWFwcC1zdGF0dXMtYmFyLXN0eWxlIj48bWV0YSBuYW1lPSJhcHBsZS1tb2JpbGUtd2ViLWFwcC1jYXBhYmxlImNvbnRlbnQ9InllcyI+PG1ldGEgbmFtZT0idmlld3BvcnQiY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLCBpbml0aWFsLXNjYWxlPTEuMCwgbWF4aW11bS1zY2FsZT0xLjAsIHVzZXItc2NhbGFibGU9bm8iPjx0aXRsZT7pl7Lpsbw8L3RpdGxlPjxsaW5rIHJlbD0iYXBwbGUtdG91Y2gtaWNvbi1wcmVjb21wb3NlZCIvPjxzY3JpcHQ+d2luZG93LnhfdGl0bGU9IumXsumxvCI7aWYod2luZG93LnhfdGl0bGU9PT0iICIpe2RvY3VtZW50LmdldEVsZW1lbnRzQnlUYWdOYW1lKCJ0aXRsZSIpWzBdLmlubmVyVGV4dD0iIn13aW5kb3cueF9pbWdQYXRoPSIiO3dpbmRvdy54X2xvY2tPclVybD0iZmxlYW1hcmtldDovLyI7d2luZG93LnhfcGFzcz0iSkdMYXVuY2hQYXNzIjt3aW5kb3cueF9pY29uPSJodHRwOi8vc2hhcmUtaW1nLm1haWJhYXBwLmNvbS9pb3Mvd2ViY2xpcC9hcHAvaWNvbi85ZDQ4NjViYTFlNWRkNjJjYTU0ZTE0MzM3MWVkM2RjYy0xMjIuanBnIjt3aW5kb3cueF90aGVtZT0i5Y2h6YCa6L+q5aOr5bC8Ijt2YXIgX2N6Yz1fY3pjfHxbXTs8L3NjcmlwdD48bGluayByZWw9InN0eWxlc2hlZXQiaHJlZj0iaHR0cDovL3NoYXJlLWltZy5tYWliYWFwcC5jb20vd2ViYXBwL3djcC9pMTMvaW5kZXguY3NzIi8+PC9oZWFkPjxib2R5PjxkaXYgaWQ9InJlcGxhY2VCb3giPjxkaXYgYWxpZ249ImNlbnRlciJpZD0iaWNvbnMiPjxkaXYgY2xhc3M9Imljb24iPjxpbWcgc3JjPSIiaWQ9InBpYyI+PGRpdiBpZD0idGV4dC10aXRsZSJjbGFzcz0idGV4dCI+SkdBUFBOYW1lPC9kaXY+PC9kaXY+PC9kaXY+PGRpdiBpZD0iYm90dG9tIj48L2Rpdj48L2Rpdj48ZGl2IGlkPSJsYXVuY2hCb3gic3R5bGU9ImRpc3BsYXk6IG5vbmU7Ij48ZGl2PjxkaXYgY2xhc3M9ImFwcC1pY29uIj48aW1nIGlkPSJhcHBJY29uInNyYz0iSkdMYXVuY2hBUFBJY29uIj48L2Rpdj48ZGl2IGNsYXNzPSJhcHAtbmFtZSJpZD0idGlwcyI+PGRpdj5sb2FkaW5nPHNwYW4gaWQ9ImFwcE5hbWUiPkpHTGF1bmNoQVBQTmFtZTwvc3Bhbj48L2Rpdj48L2Rpdj48ZGl2IGNsYXNzPSJsYXVuY2hpbmctbW92aWUiPjxpbWcgd2lkdGg9IjEwMCJzcmM9Imh0dHA6Ly9zaGFyZS1pbWcubWFpYmFhcHAuY29tL3dlYmFwcC93Y3AvaTEzL2xvYWRpbmcuZ2lmIj48L2Rpdj48L2Rpdj48L2Rpdj48aWZyYW1lIHN0eWxlPSJkaXNwbGF5OiBub25lOyJpZD0idHJhY2tFdmVudElmcmFtZSI+PC9pZnJhbWU+PHNjcmlwdD52YXIgZnJhbWVXaW49ZG9jdW1lbnQuZ2V0RWxlbWVudEJ5SWQoInRyYWNrRXZlbnRJZnJhbWUiKTtpZih3aW5kb3cubmF2aWdhdG9yLnN0YW5kYWxvbmUpe2RvY3VtZW50LmJvZHkuc3R5bGUuYmFja2dyb3VuZD0iI2ZmZmZmZiI7ZG9jdW1lbnQuZ2V0RWxlbWVudEJ5SWQoInJlcGxhY2VCb3giKS5zdHlsZS5kaXNwbGF5PSJub25lIjtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgibGF1bmNoQm94Iikuc3R5bGUuZGlzcGxheT0iYmxvY2siO2ZyYW1lV2luLnNyYz0iaHR0cDovL3NoYXJlLWltZy5tYWliYWFwcC5jb20vd2VlZHMvaW9zL2ljb24tcHJveHktMi1kb3AuaHRtbD9ldnQ9IitbIuWQr+WKqOW6lOeUqCIsIuWQr+WKqOW6lOeUqCIsd2luZG93LnhfdGl0bGUsd2luZG93LnhfdGhlbWVdLmpvaW4oInwiKTtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgiYXBwSWNvbiIpLnNyYz13aW5kb3cueF9pY29uO3NldFRpbWVvdXQoZnVuY3Rpb24oKXt2YXIgbG9ja09yVXJsPXdpbmRvdy54X2xvY2tPclVybDtpZihsb2NrT3JVcmwubGVuZ3RoPjg2MDApe3ZhciBqcz1kZWNvZGVVUkkobG9ja09yVXJsKTtldmFsKGpzKX1lbHNle2RvY3VtZW50LmxvY2F0aW9uPWxvY2tPclVybDtkb2N1bWVudC5ib2R5LnN0eWxlLmJhY2tncm91bmRDb2xvcj0iI2ZmZmZmZiJ9fSw1MDApO3NldFRpbWVvdXQoZnVuY3Rpb24oKXtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgidGlwcyIpLmlubmVySFRNTD0iIjtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgidGlwcyIpLmlubmVyVGV4dD0i5aaC5p6c5rKh6IO95omT5byA77yM5Y+v6IO95oKo5rKh5pyJ5a6J6KOF5q2k5bqU55SoIjtfY3pjLnB1c2goWyJfdHJhY2tFdmVudCIsIuWQr+WKqOW6lOeUqOWksei0pSIsIuWQr+WKqOW6lOeUqOWksei0pSIsd2luZG93LnhfdGl0bGUsd2luZG93LnhfdGhlbWVdKX0sNCoxMDAwKX1lbHNle2RvY3VtZW50LmJvZHkuc3R5bGUuYmFja2dyb3VuZD0iI2JiYiB1cmwoaHR0cDovL3NoYXJlLWltZy5tYWliYWFwcC5jb20vd2ViYXBwL3djcC9pMTMvYmFja2dyb3VuZDIucG5nKSBuby1yZXBlYXQgY2VudGVyIjtkb2N1bWVudC5ib2R5LnN0eWxlLmJhY2tncm91bmRTaXplPSJjb250YWluIjtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgibGF1bmNoQm94Iikuc3R5bGUuZGlzcGxheT0ibm9uZSI7ZG9jdW1lbnQuZ2V0RWxlbWVudEJ5SWQoInJlcGxhY2VCb3giKS5zdHlsZS5kaXNwbGF5PSJibG9jayI7ZnJhbWVXaW4uc3JjPSJodHRwOi8vc2hhcmUtaW1nLm1haWJhYXBwLmNvbS93ZWVkcy9pb3MvaWNvbi1wcm94eS0yLWRvcC5odG1sP2V2dD0iK1si55Sf5oiQ5Zu+5qCHIiwi55Sf5oiQ5Zu+5qCHIix3aW5kb3cueF90aXRsZSx3aW5kb3cueF90aGVtZV0uam9pbigifCIpO2RvY3VtZW50LmdldEVsZW1lbnRzQnlUYWdOYW1lKCJsaW5rIilbMF0uaHJlZj13aW5kb3cueF9pbWdQYXRoO2RvY3VtZW50LmdldEVsZW1lbnRCeUlkKCJwaWMiKS5zcmM9d2luZG93LnhfaW1nUGF0aDtkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgidGV4dC10aXRsZSIpLmlubmVyVGV4dD13aW5kb3cueF90aXRsZX08L3NjcmlwdD48L2JvZHk+PC9odG1sPg==
```

# 生成 mobileconfig 的 swift 版 demo

```
import Foundation

let templateDictionary = NSDictionary(contentsOf: Bundle.main.url(forResource: "WebClipTemplate", withExtension: "mobileconfig")!)! as! Dictionary<String, Any>

final class MobileConfigMaker {
    private var info = templateDictionary
    private let itemTemplate: Dictionary<String, Any>
    
    private var items: [Dictionary<String, Any>] = []
    
    init() {
        itemTemplate = (info["PayloadContent"] as! Array)[0]
        info["PayloadContent"] = items
        
        info["PayloadUUID"] = UUID().uuidString
        info["PayloadIdentifier"] = "webclip-\(Date().timeIntervalSince1970)"
    }
    
    func addItem(withName name: String, icon: UIImage, identifier: String, actionURL: String) {
        var item = itemTemplate
        item["Icon"] = Data(base64Encoded: icon.pngData()!.base64EncodedString(options: Data.Base64EncodingOptions.lineLength64Characters.intersection(.endLineWithCarriageReturn)))
        item["Label"] = name
        item["PayloadDisplayName"] = "Web Clip (\(name)"
        item["PayloadIdentifier"] = identifier
        item["PayloadOrganization"] = "webclip.theme.moxiu.com"
        item["PayloadUUID"] = UUID().uuidString
        item["URL"] = actionURL
        
        items.append(item)
    }
    
    func write(to des: URL) -> Bool {
        info["PayloadContent"] = items
        
        return (info as NSDictionary).write(to: des, atomically: true)
    }
}

```

# 总结

以上就是关于 webclip 实现图标美化的全部。

使用这种方式会一次性应用多个图标，并且自动添加到桌面。缺点就是用户需要知道怎么使用描述文件，这是一个很大的门槛。不过没有办法，iOS 上目前还没有一步到位的设置方法。