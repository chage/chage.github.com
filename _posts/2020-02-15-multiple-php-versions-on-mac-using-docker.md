---
layout: post
title: 在 Mac 上透過 docker 來執行多版本的 PHP
---
因為一些緣故要在 Mac 上安裝 PHP 5.6 版，由於在 Homebrew 上已經無法安裝了，所以只好想點別的辦法。  
而且要避免日後系統更新又被蓋掉。  
本來的環境是 apache -&gt; mod_php / php-fpm -&gt;  \*.php  
於是想試試看用 docker 來跑 php-fpm 部分，查了一下還蠻多人這樣做的。  

## 1.安裝 docker  
可以參考[官方文件](https://docs.docker.com/docker-for-mac/)  
我是用 Homebrew 的方式安裝，不過這只是把下載 Docker.dmg 包起來的樣子。  
```
brew cask install docker
```
裝完應該立刻就能用了  

## 2.安裝 PHP 的 docker 版  
這個部分有蠻多人打包方便或是比較小的版本，不過我比較喜歡裝[官方的版本](https://hub.docker.com/_/php/)  
```
docker pull php:5.6-fpm
docker create --name php56 -p 9056:9000 -v /Users/chage/project:/Users/chage/project php:5.6-fpm
docker start php56
```
或者也可以直接讓它下載執行
```
$ docker run -d --name php56 -p 9056:9000 -v /Users/chage/project:/Users/chage/project php:5.6-fpm
Unable to find image 'php:5.6-fpm' locally
5.6-fpm: Pulling from library/php
...
```
由於我的 host 也有跑其他版本的 php-fpm 在預設的 port 9000，所以改為用 9056 對應到 docker 裡的 9000。  
另外，我的程式放在 `/Users/chage/project` ，必須加到 docker 的 volume 裡，要不然 php-fpm 會找不到檔案。  

## 3.Dockfile  
實際跑了以後，發現少了一些 extension，所以只好回頭安裝。  
看來看去還是用 Dockerfile 自己建立一版比較清楚一點。  
```
FROM php:5.6-fpm

RUN apt-get update

# Install PostgreSQL PDO
RUN apt-get install -y libpq-dev \
    && docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
    && docker-php-ext-install pdo pdo_pgsql pgsql

RUN docker-php-ext-install mysqli pdo_mysql

RUN apt-get install -y libzip-dev
RUN docker-php-ext-install zip

RUN apt-get install -y libjpeg62-turbo-dev libpng-dev libxpm-dev \
    libfreetype6-dev \
    && docker-php-ext-configure gd --with-gd --with-webp-dir --with-jpeg-dir \
    --with-png-dir --with-zlib-dir --with-xpm-dir --with-freetype-dir \
    --enable-gd-native-ttf \
    && docker-php-ext-install gd

RUN apt-get install -y libxml2-dev
RUN docker-php-ext-install soap

RUN docker-php-ext-install sockets
```
之後執行  
```
docker build -t chage/php:5.6-fpm
docker run -d --name php56 -p 9056:9000 -v /Users/chage/project:/Users/chage/project chage/php:5.6-fpm
```

## 4.修改 apache / nginx  
由於 docker 裡的 php-fpm 跑在 port 9056，所以要修改 webserver 的設定。  
apache / nginx 把本來 php-fpm 的 port 改為 9056。  
nginx 如下:  
```
...
location ~ \.php$ {
  #fastcgi_pass 127.0.0.1:9000;
  fastcgi_pass 127.0.0.1:9056;
  include /etc/nginx/fastcgi_params;
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
...
```
apache 如下:  
```
...
<FilesMatch \.php$>
    #SetHandler proxy:fcgi://localhost:9000
    SetHandler proxy:fcgi://localhost:9056
</FilesMatch>
...
```
改完再重啟 apache / nginx 服務就好了  

## 5.其他  
* apache 從 mod_php 轉換為 php-fpm 時，容易遇到 `.htaccess` 裡寫了的 `php_value` 不吃而爛掉的狀況，要記得改為 `.user.ini` 的版本。我目前還沒找到方便 mod_php 跟 php-fpm 都吃的解決法，這點有點困擾。  
* 由於我的主機上跑了不少網站，為了讓 container 裡連到 host 上的網站，要在啟動的地方加上 `--add-host` 參數來新增 `/etc/hosts` 的對應，在 mac 下可以用 ``--add-host=site1.local:`ipconfig getifaddr en0` --add-host=site2.local:`ipconfig getifaddr en0` ... `` 來新增。  

