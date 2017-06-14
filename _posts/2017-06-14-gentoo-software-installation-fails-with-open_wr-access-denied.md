---
layout: post
title: Gentoo Software Installation Fails With open_wr ACCESS DENIED
---
實在不該隔太久沒更新系統的，發生好多問題。  
這次更新系統時，在安裝 `app-i18n/ibus-chewing` 卡了很久，發生沒遇過的狀況。  
（不過整個系統更新的過程有一堆狀況就是了，Gentoo Linux 的日常...Orz）  

出現的訊息像這樣：  

```
...
[Info1] CMAKE_HOST_SYSTEM=Linux-4.0.6-gencool
[Info1] CMAKE_HOST_SYSTEM_PROCESSOR=x86_64
[Info1] CMAKE_SYSTEM=Linux-4.0.6-gencool
[Info1] CMAKE_SYSTEM_PROCESSOR=x86_64
-- Found PkgConfig: x86_64-pc-linux-gnu-pkg-config (found version "0.28")
 * ACCESS DENIED:  open_wr:      /var/lib/rpm/__db.001
[Error] Package cmake is not installed
 * ACCESS DENIED:  open_wr:      /var/lib/rpm/__db.001
[Error] Package ibus is not installed
 * ACCESS DENIED:  open_wr:      /var/lib/rpm/__db.001
[Error] Package ibus-devel is not installed
...
```

原本以為是少裝了什麼，下錯關鍵字，所以都查不到類似資料。  
只好跟 cooldavid 大神求救。  
結果原來是 sandbox 裡的權限，中間繞了不少路啊...  

[Knowledge Base:Software installation fails with open wr ACCESS DENIED on SELinux systems](https://wiki.gentoo.org/wiki/Knowledge_Base:Software_installation_fails_with_open_wr_ACCESS_DENIED_on_SELinux_systems)  

在 `/etc/sandbox.conf` 加上：  

```
...
SANDBOX_WRITE="/var/lib/rpm"
```

再次 emerge 就過了。（泣）  
記錄一下，免得日後又得再繞一次路。  
再次覺得有 cooldavid 真好～（心）  
