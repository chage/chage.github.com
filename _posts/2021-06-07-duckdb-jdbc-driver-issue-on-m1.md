---
layout: post
title: DuckDB JDBC Driver Issue on Macbook M1
---
最近在 Macbook M1 上開發 Clojure 時遇到一個問題。  
當我試圖去查詢 DuckDB 時，我的 REPL 出現以下訊息：  

```
user=> java.io.IOException: /libduckdb_java.so_osx_ not found
	at org.duckdb.DuckDBNative.<clinit>(DuckDBNative.java:36)
	at org.duckdb.DuckDBDatabase.<init>(DuckDBDatabase.java:22)
	at org.duckdb.DuckDBDriver.connect(DuckDBDriver.java:35)
	at java.sql/java.sql.DriverManager.getConnection(DriverManager.java:677)
	at java.sql/java.sql.DriverManager.getConnection(DriverManager.java:189)
	at next.jdbc.connection$get_driver_connection.invokeStatic(connection.clj:141)
	at next.jdbc.connection$get_driver_connection.invoke(connection.clj:136)
	at next.jdbc.connection$url_PLUS_etc$reify__32262.getConnection(connection.clj:359)
	at next.jdbc.connection$make_connection.invokeStatic(connection.clj:385)
	at next.jdbc.connection$make_connection.invoke(connection.clj:369)
	at next.jdbc.connection$eval32281$fn__32282.invoke(connection.clj:408)
	at next.jdbc.protocols$eval32038$fn__32039$G__32029__32046.invoke(protocols.clj:24)
	at next.jdbc.result_set$eval32979$fn__32984.invoke(result_set.clj:896)
	at next.jdbc.protocols$eval32070$fn__32071$G__32059__32080.invoke(protocols.clj:33)
	at next.jdbc$execute_one_BANG_.invokeStatic(jdbc.clj:263)
	at next.jdbc$execute_one_BANG_.invoke(jdbc.clj:250)
	...
```
查了一些文章後有點絕望，因為看起來是 DuckDB 的 JDBC driver 不支援 M1。  
比如說[這裡提到的](https://github.com/duckdb/duckdb/issues/941)，但看來沒下文。  
  
掙扎了很久，最後把心一橫，想說抓 source code 下來編看看。  
```
$ git clone https://github.com/duckdb/duckdb.git
$ cd duckdb
$ export JAVA_HOME=/opt/homebrew/Cellar/openjdk/15.0.2/libexec/openjdk.jdk/Contents/Home
$ BUILD_JDBC=1 make
```
編完後，可以在 `build/release/tools/jdbc/` 找到 `duckdb_jdbc.jar` 跟 `libduckdb_java.so_osx_amd64`  
由於前面 REPL 的訊息寫著 `/libduckdb_java.so_osx_ not found`  
所以就試著去找出哪邊組這串的，最後在 `tools/jdbc/src/main/java/org/duckdb/DuckDBNative.java` 裡找到。  
```
            String os_name = "";
            String os_arch = "";
            String os_name_detect = System.getProperty("os.name").toLowerCase().trim();
            String os_arch_detect = System.getProperty("os.arch").toLowerCase().trim();
            if (os_arch_detect.equals("x86_64") || os_arch_detect.equals("amd64")) {
                os_arch = "amd64";
            }
            // TODO 32 bit gunk

            if (os_name_detect.startsWith("windows")) {
                os_name = "windows";
            } else if (os_name_detect.startsWith("mac")) {
                os_name = "osx";
            } else if (os_name_detect.startsWith("linux")) {
                os_name = "linux";
            }
            String lib_res_name = "/libduckdb_java.so" + "_" + os_name + "_" + os_arch;
```
所以應該是 `os_arch_detect` 不知道得到了什麼（雖然大概能猜到），做了一下實驗，得到是 `aarch64`  
由於稍早編譯完後得到的 `libduckdb_java.so_osx_amd64` 檔名似乎不會變，所以試著讓這段的 `lib_res_name` 最後組成一樣的檔名。  
將上面那段加上一段 `aarch64` 的部分：  
```
            String os_name = "";
            String os_arch = "";
            String os_name_detect = System.getProperty("os.name").toLowerCase().trim();
            String os_arch_detect = System.getProperty("os.arch").toLowerCase().trim();
            if (os_arch_detect.equals("x86_64") || os_arch_detect.equals("amd64")) {
                os_arch = "amd64";
            }
			if (os_arch_detect.equals("aarch64")) {
                os_arch = "amd64";
            }
            // TODO 32 bit gunk

            if (os_name_detect.startsWith("windows")) {
                os_name = "windows";
            } else if (os_name_detect.startsWith("mac")) {
                os_name = "osx";
            } else if (os_name_detect.startsWith("linux")) {
                os_name = "linux";
            }
            String lib_res_name = "/libduckdb_java.so" + "_" + os_name + "_" + os_arch;
```
重新再編譯一次後，得到新的 `duckdb_jdbc.jar` 更名放至 `~/.m2/repository/org/duckdb/duckdb_jdbc/0.2.5/duckdb_jdbc-0.2.5.jar`  
（我的環境抓的 duckdb driver 版號是 0.2.5，不過我是抓最新的程式來編）  
然後重啟 REPL 就可以查詢了，喔耶～  
  
終於可以開始工作了……（攤）  
