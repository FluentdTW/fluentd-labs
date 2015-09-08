# Lab 1 - Fluentd 起手式

想學習 [Fluentd](http://www.fluentd.org) 嗎？ [Docker](http://www.docker.com) 是您的好朋友!!

感謝 [William Yeh 葉秉哲](https://twitter.com/williamyeh) 製作的 [fluentd (td-agent) docker image](https://hub.docker.com/r/williamyeh/fluentd/)，內含 [fluentd](http://www.fluentd.org) 與 [fluentd-ui](http://github.com/fluent/fluentd-ui)

以下透過三個 GIF 動畫簡單展示一下 Fluentd 的 in_http 與 out_stdout 兩個 plugin

--------

**圖 1 : 使用 williamyeh/fluentd docker image 帶起 fluentd-ui**

![使用 williamyeh/fluentd docker image 帶起 fluentd-ui](figures/day1_fluentd-ui_1.gif)

## STEP 1

首先，使用 `williamyeh/fuentd` 這個 image 啟動一個名為 `fluentd-ui` 的 docker instance，並使用 `fluentd-ui start` 做為啟動參數

```
$ docker run -d --name fluentd-ui williamyeh/fluentd fluentd-ui start
```

## STEP 2

檢查是否正常開啟於 9292 連線埠

```
$ docker logs fluentd-ui
```

## STEP 3

使用指令檢查 fluentd-ui 的 IP 位址

```
$ docker exec fluentd-ui hostname -I
```

--------

**圖 2 : 登入 Fluentd UI 並啟動 Fluentd**

![登入 Fluentd UI 並啟動 Fluentd](figures/day1_fluentd-ui_2.gif)

## STEP 4

小撇步：使用指令產生 fluentd-ui 的 URL

```
$ echo "http://$(docker exec fluentd-ui hostname -I | awk '{ print $1 }'):9292"
```

點選產生的 URL 就可以看到 Fluentd 的網頁介面

## STEP 5

1. 使用 fluentd-ui 預設帳密登入 `admin` / `changeme`
2. 選擇 `Setup Fluentd` 然後按下 `Create` 按鈕，使用預設值將 Fluentd 的設定檔與日誌檔存放於 /root/.fluentd-ui/ 目錄下。
3. 按下 `Start` 按鈕，啟動 Fluentd 的背景程式。

--------

**圖 3 : 登入 Fluentd UI 並啟動 Fluentd**

![使用 williamyeh/fluentd docker image 帶起 fluentd-ui](figures/day1_fluentd-ui_3.gif)


## STEP 6

以下示範如何送資料到 fluentd 的 `in_http` plugin

以下是在 Ubuntu 底下，使用 curl 送出 POST 到 fluentd daemon 的 9880 連線埠，JSON 內容為 { "action":"login", "user": 2 }

您也可以透過瀏覽器的網頁開發者介面或 [Firefox RESTful Client](https://addons.mozilla.org/en-Us/firefox/addon/restclient/) . [Chrome RESTful Client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo) 進行以下動作：

```
$ IP=$(docker exec fluentd-ui hostname -I | awk '{ print $1 }')
$ curl -i -X POST -d 'json={"action":"login","user":2}' http://$IP:9880/debug.http
```

這裡送到 9880 連線埠，是因為 fluentd-ui 預設產生的設定檔 fluent.conf 包含以下這幾行：

```
  <source>
    type http
    port 9880
  </source>
```

## STEP 7

檢查 fluentd daemon 的 out_stdout 輸出

```
$ docker exec fluentd-ui cat /root/.fluentd-ui/fluent.log
```

上面的 POST 送的路徑有玄機，您可以試著將原網址的 `debug.http` 改變 debug 後面接的文字，如 debug.**test** 看看 STDOUT 的結果會有什麼變化。

為什麼要接 debug 這個關鍵字呢？那是因為 fluent.conf 設定檔裏面所設定的 match 規則，是檢查 `debug.*` 這樣的「規則(Rule)」

```
<match debug.*>
  # http://docs.fluentd.org/articles/out_stdout
  type stdout
</match>
```

因此如果您送給 in_http 的 URL 不是 debug.* 那麼在 out_stdout 就不會出現結果哦~~

例如執行以下指令，並結果不會出現 {"action":"search","user":3}' 字樣：

```
$ IP=$(docker exec fluentd-ui hostname -I | awk '{ print $1 }')
$ curl -i -X POST -d 'json={"action":"search","user":3}' http://$IP:9880/new.message
$ docker exec fluentd-ui tail -n 5 /root/.fluentd-ui/fluent.log
```

## 後記

有時候 Docker 第一次開正常，但不正常關閉或者用 `docker stop fluentd-ui` 關閉後，想要再開就會開不起來。那怎麼查原因呢？

可以透過 `docker logs fluentd-ui` 這個指令來查。舉例來說，透過 `docker logs fluentd-ui` 觀察到以下的現象：

```
2015-09-08 07:07:48 +0000: Read error: #<Errno::ENOTCONN: Transport endpoint is not connected - getpeername(2)>
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/server.rb:460:in `peeraddr'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/server.rb:460:in `normalize_env'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/server.rb:486:in `handle_request'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/server.rb:375:in `process_client'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/server.rb:262:in `block in run'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/thread_pool.rb:104:in `call'
/opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/puma-2.11.0/lib/puma/thread_pool.rb:104:in `block in spawn_thread'
A server is already running. Check /opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-ui-0.3.18/tmp/fluentd-ui.pid.
A server is already running. Check /opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-ui-0.3.18/tmp/fluentd-ui.pid.
```

代表 fluentd-ui 因為不正常關閉，因此 fluentd-ui.pid 這個檔案還存在。

那如何修復呢？有時候要搶速度，只好把兩個指令放在一起跑囉：

```
~$ docker start fluentd-ui ; docker exec fluentd-ui rm /opt/td-agent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-ui-0.3.18/tmp/fluentd-ui.pid
fluentd-ui
```

通常如果正常移除 pid 檔，就可以順利重新開啟 fluentd-ui 這個 Docker instance 囉~

```
$ docker ps -a
CONTAINER ID        IMAGE                COMMAND              CREATED             STATUS                    PORTS                            NAMES
b483b6c72fae        williamyeh/fluentd   "fluentd-ui start"   7 hours ago         Up 15 seconds             9880/tcp, 24224/tcp, 24230/tcp   fluentd-ui
```