---
layout: post
title: Upgrade the Kernel of Gentoo Linux on Linode
---
由於最近 Gentoo Linux [公告](https://www.gentoo.org/support/news-items/2024-03-22-new-23-profiles.html)說舊的 Profile 如 17.0、17.1、20.0、22.0 會被 deprecated，要升到 23.0。但我的主機是放在 Linode 上，所以一直怕怕的，畢竟摸不到主機。不過後來想起 Linode 現在有 Backup 服務，所以就把心一橫衝了。然後不知道哪來的勇氣，也順便把卡在 4.14 的 kernel 升級到目前的 6.6。  

當然首先就是先花錢啟用 Backup，不過其實不怎麼貴。然後手動執行一次備份。  

事實上升級 Profile 還算順利，我的 Profile 本來是 17.1，所以少了一些步驟要做，照著文件很順利地跑到了最後。（當然這中間斷斷續續跑了幾天 Orz）  
升級完後，系統還算正常，上面跑的服務也不多，而且都是自己用，所以都是自己的專案裡一些像 venv 指到的 Python path 找不到（因為 3.11 升級成 3.12），或 Java 被升級造成 Clojure 專案要加個 jvm-opts，不過錯誤訊息有寫好怎麼處理，所以解的蠻快的。  

```
java.lang.RuntimeException: Unable to load direct buffer constructor.  If you are using JDK-17, set your runtime :jvm-opts as follows:
:jvm-opts ["--add-modules" "jdk.incubator.foreign,jdk.incubator.vector"
                         "--enable-native-access=ALL-UNNAMED"]}
```

這都是因為系統軟體升級造成的，跟 Profile 升級沒什麼關係就是，哈哈哈。  
接下來才是我想寫這篇的主因，怕日後有需要升級，留著給自己參照。  

我的主機是 2019 年建立的，當時 Linode 提供的 kernel 是 4.14.83，就這樣用了很多年不敢升級。一部分的原因是因為當年還沒有 Backup 服務，不過隔年公告了這個服務後，我也沒立刻就開始升級，我想我實在是太害怕了（默）。  
由於為了升級 Profile ，有種系統變的很乾淨的錯覺，就想順便把 kernel 也升一升，讓這些麻煩事做一次工。  

首先當然還是再按一次備份。XDDD  

先查看了一下目前有什麼可以選。  

```
# eselect kernel list
Available kernel symlink targets:
  [1]   linux-4.14.83-gentoo *
  [2]   linux-6.6.38-gentoo
```

然後參照[官方文件](https://wiki.gentoo.org/wiki/Kernel/Upgrade)的說明，設定為最新的。  

```
# eselect kernel set 2
```

這步會讓 `/usr/src/linux` 指到新的那個。如下：  

```
# ls -l /usr/src/linux
lrwxrwxrwx 1 root root 19 Jul 27 14:46 /usr/src/linux -> linux-6.6.38-gentoo
```

然後把舊的 .config 複製到目前這個新的目錄下。  
這部分文件裡寫了不少方式，總之我以下這兩個的結果是相同的內容：  

* `zcat /proc/config.gz`
* `/etc/kernels/kernel-config-x86_64-4.14.83-gentoo`

所以就直接選這兩個其中一種  
```
# cp /etc/kernels/kernel-config-x86_64-4.14.83-gentoo /usr/src/linux/.config
```

因為這個 .config 是舊的，要更新成新的才行真的拿來編譯 kernel。這個部分，文件上寫了兩種方式，一種有點算是全手動，一種是讓它的 script 從更新到編譯整個跑完。  
不過我一開始看文件不是很確定它的意思，總之就先全手動一步一步來。  

全手動的流程是：  

```
# cd /usr/src/linux
# make oldconfig
# make modules_prepare
# make
# emerge --ask @module-rebuild
# make modules_install
# make install
```

不過裡面的 `make oldconfig` 因為會一個一個選，我後來就改用 `make olddefconfig`，它會將新的設定都用 default 值。  

```
/usr/src/linux # make olddefconfig
.config:533:warning: symbol value 'm' invalid for I8K
.config:1000:warning: symbol value 'm' invalid for NF_CT_PROTO_GRE
.config:1030:warning: symbol value 'm' invalid for NF_TABLES_INET
.config:1207:warning: symbol value 'm' invalid for NF_TABLES_IPV4
.config:1249:warning: symbol value 'm' invalid for NF_TABLES_IPV6
.config:1752:warning: symbol value 'm' invalid for MTD_NAND_ECC
.config:2941:warning: symbol value 'm' invalid for ISDN_CAPI
```

終於編完後，然後就是裝到 bootloader 裡，我的是 GRUB 2。  

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

然後 reboot 後，它就 Kernel Panic 了…… Q_Q  
從 LISH console 看到以下訊息：  

```
[    1.335926] Kernel panic - not syncing: VFS: Unable to mount root fs on unkn)
[    1.336625] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 6.6.38-gentoo #1
[    1.337155] Hardware name: Linode Compute Instance, BIOS Not Specified
[    1.337711] Call Trace:
[    1.337937]  <TASK>
[    1.338139]  dump_stack_lvl+0x4a/0x80
[    1.338468]  panic+0x199/0x360
[    1.338763]  mount_root_generic+0x29e/0x300
[    1.339145]  prepare_namespace+0x69/0x280
[    1.339525]  kernel_init_freeable+0x1a2/0x1f0
[    1.339898]  ? __pfx_kernel_init+0x10/0x10
[    1.340235]  kernel_init+0x1a/0x1b0
[    1.340546]  ret_from_fork+0x34/0x50
[    1.340852]  ? __pfx_kernel_init+0x10/0x10
[    1.341194]  ret_from_fork_asm+0x1b/0x30
[    1.341524]  </TASK>
[    1.341779] Kernel Offset: 0x13200000 from 0xffffffff81000000 (relocation ra)
[    1.342665] ---[ end Kernel panic - not syncing: VFS: Unable to mount root f-
```

（哈利路亞，嗆思～）  

總之先 restore Backup 回到了 Profile 剛升級完後，然後爬文找一下有沒有什麼人有類似經驗的。不過要找到一樣情境的實在太難了，到底誰會特地在雲端主機上裝 Gentoo Linux 呢？（話說 Linode 一直有這個 OS 的選項，所以也許真的有人在用，也希望他們能保持下去）  
但我找到以下兩篇是有相關的：  

* [Guides - Manage the Kernel on a Compute Instance](https://www.linode.com/docs/products/compute/compute-instances/guides/manage-the-kernel/)
* [Tutorial: Upgrade a small Gentoo Instance on Linode](https://www.reddit.com/r/linux/comments/11nj1dh/tutorial_upgrade_a_small_gentoo_instance_on_linode/)

看起來都跟我做的相同，在試過一些設定上的差異還是失敗後，我想到我還有一個方式沒試過，那就是 genkernel。  
前面有提到，其實文件有說到兩個方式去更新跟編譯 kernel，另一個我在看了上面兩篇後，才比較確定它的用法。  
當然還是要先看一下 [genkernel 的文件](https://wiki.gentoo.org/wiki/Genkernel)，看 oldconfig 參數的作用是什麼。  
最後我是改用以下方式更新 .config 並編譯 kernel，然後安裝到 GRUB 2：  

```
# cd /usr/src/linux
# genkernel --oldconfig all
# grub-mkconfig -o /boot/grub/grub.cfg
```

這裡我注意到有這個訊息（這個手動時也會出現，只是我放在這裡一起說）：  

```
...
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
...
```

依照 [GRUB 文件 - Troubleshooting](https://wiki.gentoo.org/wiki/GRUB#os-prober_not_running) 裡寫的，這個要去 `/etc/default/grub` 把該設定關掉。

```
GRUB_DISABLE_OS_PROBER=false
```

再跑一次 `grub-mkconfig -o /boot/grub/grub.cfg` 就會正常了。  
這次 reboot 後就順利地正常開機，喔耶！！🎉  

觀察手動跟自動的檔案差異，我留意到在 `/boot` 下本來（好像）不會有 `initramfs-6.6.38-gentoo-x86_64.img`，但現在有了。  
但可以看到 Linode 自己放的 4.14 版有，而且有 genkernel 字樣，所以，這個流程應該是對的了。  

```
# ls -alh /boot/
total 41M
drwxr-xr-x  3 root root 4.0K Jul 27 16:44 .
drwxr-xr-x 21 root root 4.0K Jul 27 01:26 ..
-rw-r--r--  1 root root  76K Jul 26 15:16 amd-uc.img
drwxr-xr-x  6 root root 4.0K Jul 27 22:27 grub
-rw-r--r--  1 root root  11M Jul 27 16:43 initramfs-6.6.38-gentoo-x86_64.img
-rw-r--r--  1 root root 7.3M Jan  3  2019 initramfs-genkernel-x86_64-4.14.83-gentoo
-rw-r--r--  1 root root    0 Jan  2  2019 .keep
-rw-r--r--  1 root root 5.9M Jan  3  2019 kernel-genkernel-x86_64-4.14.83-gentoo
-rw-r--r--  1 root root 5.3M Jul 27 15:14 System.map-6.6.38-gentoo-x86_64
-rw-r--r--  1 root root 2.9M Jan  3  2019 System.map-genkernel-x86_64-4.14.83-gentoo
-rw-r--r--  1 root root 9.0M Jul 27 15:14 vmlinuz-6.6.38-gentoo-x86_64
```

另外，`/boot/grub/grub.cfg` 裡面的 initrd 多了一段 `/boot/amd-uc.img`，不過這個應該是因為系統更新所以要加上去的，跟 kernel 更新失敗無關（？）  

```
                initrd  /boot/amd-uc.img /boot/initramfs-6.6.38-gentoo-x86_64.img
```

總之這樣日後要再更新就比較不會怕了（總之先按下 backup XD）  
