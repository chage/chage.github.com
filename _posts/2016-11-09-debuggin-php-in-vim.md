---
layout: post
title: Debuging PHP in Vim
---
寫 PHP 很久了，不過一直以來除錯都是透過 echo、var_dump 等方式（掩面）。  
雖然早就聽過 Xdebug，不過一直就沒空試。最近看了文件，想說就來試著裝一裝吧。  
大致上就是安裝 Xdebug，然後設定 PHP，最後是讓 Vim 接上 Xdebug  
其實應該要很順，不過我裝的時候遇到一點狀況，以致於弄了一陣子才搞懂。  
  
Xdebug  
  
我用的是 Mac，所以就照[這裡](https://xdebug.org/docs/install#mac "Installation on Mac OS X via Homebrew")安裝 Xdebug  
其實就只是：  
`# brew install <php-version>-xdebug`  
例如我的是  
`# brew install php56-xdebug`  
  
PHP  
  
Xdebug 安裝完後就會在 `/usr/local/etc/php/<php-version>/conf.d/` 有個 `ext-xdebug.ini`  
在裡面加上 Xdebug 的參數設定  
```
[xdebug]
zend_extension="/usr/local/opt/php56-xdebug/xdebug.so"
xdebug.remote_autostart=1
xdebug.remote_enable=1
xdebug.remote_host=localhost
xdebug.remote_port=9000
```
remote_autostart 是讓 request 進來時，都會去觸發 Xdebug。  
如果不設 autostart，那就要在 request 後面加上 `?XDEBUG_SESSION_START=1` 呼叫一次。這會讓 Xdebug 知道這個連線需要執行 Xdebug，並記錄到 cookie 裡。  
remote_host 這裡設 localhost，是因為我在本機跑。  
remote_port 預設似乎是 9000，不過也可以更改。  
這兩項讓 Xdebug 知道要連到哪邊去取得除錯的動作。（應該是這樣理解的吧 Orz）  
編輯完後要重啟 apache  
  
Vim  
  
[Vdebug](https://github.com/joonty/vdebug/) 是 Vim 的 plugin，支援 DBGP。  
我用的 plugin manager 是 Pathogen，所以只要 `# git clone https://github.com/joonty/vdebug.git ~/.vim/bundle/vdebug` 就安裝好了。  
然後在 `.vimrc` 裡加上以下設定  
```
let g:vdebug_options = {'port': '9000'}
let g:vdebug_options = {'break_on_open': 0}
```
port 就是對應 PHP 那設定的 remote_port。  
不過我一開始設定完後，一直無沒正確觸發 Vdebug 的除錯畫面。弄了很久才發現原來是 port 9000 被我之前設定的 PHP-FPM 給佔住了 Orz  
後來把 port 移到 10000（修改 PHP 及 Vim 的設定）就好了  
break_on_open 這項設定，能讓 Vdebug 不會在 script 的第一行停下來，會停在你設的中斷點。因為預設是會一打開就先中斷，所以當用 browser 連線後，會發現網站卡住，但我們通常是希望它停在指定的地方就好。  
  
參考  
  
http://thorpesystems.com/blog/debugging-php-in-vim/  
http://www.sromero.org/wiki/linux/servicios/php/vim_debug_basics  
