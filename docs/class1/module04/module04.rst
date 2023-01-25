NGINX App Protect Dos (NAP Dos)
#######

NAP Dos について
====

NAP Dosは、ワールドワイドで実績豊富なF5製WAFのL7DoS機能を移植した、NGINX Plusの動的モジュールで実現するWAFです。

NAP DosはNGINXの動的モジュールであるという特徴から、GatewayからIngress Controller、更にコンテナとして柔軟なデプロイが可能です。

   .. image:: ./media/nap-dos-structure.png
       :width: 400

   .. image:: ./media/nap-dos-structure2.png
       :width: 400

   .. image:: ./media/nap-dos-structure3.png
       :width: 400

1. NAP Dos の設定
====

1. 設定
----

NAP Dosを設定します

.. code-block:: cmdin

  # sudo su
  cd /etc/nginx/conf.d
  cp ~/f5j-nginx-plus-lab2-security-conf/l7dos/l7dos-l1_demo.conf /etc/nginx/conf.d/default.conf
  cp ~/f5j-nginx-plus-lab2-security-conf/l7dos/l7dos-l1_plus_api.conf /etc/nginx/conf.d/plus_api.conf
  
  mkdir -p /etc/nginx/conf.d/ssl
  cp ~/f5j-nginx-plus-lab2-security-conf/ssl/* /etc/nginx/conf.d/ssl/

  # nginx.conf の copy
  cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf-
  cp ~/f5j-nginx-plus-lab2-security-conf/l7dos/l7dos-l1_nginx.conf /etc/nginx/nginx.conf


設定ファイルを確認します。 

.. code-block:: cmdin

  diff -u /etc/nginx/nginx.conf- /etc/nginx/nginx.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  --- nginx.conf- 2023-01-24 19:34:56.530164984 +0900
  +++ nginx.conf  2023-01-24 19:37:00.063823686 +0900
  @@ -8,16 +8,18 @@
  
   user  nginx;
   worker_processes  auto;
  +worker_rlimit_nofile 10240;
  
   error_log  /var/log/nginx/error.log notice;
   pid        /var/run/nginx.pid;
  
  
   events {
  -    worker_connections  1024;
  +    worker_connections 10240;
  +    accept_mutex       off;
  +    multi_accept       off;
   }
  
  -
   http {
       include       /etc/nginx/mime.types;
       default_type  application/octet-stream;
  @@ -31,28 +33,11 @@
       sendfile        on;
       #tcp_nopush     on;
  
  -    keepalive_timeout  65;
  +    keepalive_timeout  300s;
  +    keepalive_requests 1000000;
  
       #gzip  on;
  
       include /etc/nginx/conf.d/*.conf;
   }
  
  -
  -# TCP/UDP proxy and load balancing block
  -#
  -#stream {
  -    # Example configuration for TCP load balancing
  -
  -    #upstream stream_backend {
  -    #    zone tcp_servers 64k;
  -    #    server backend1.example.com:12345;
  -    #    server backend2.example.com:12345;
  -    #}
  -
  -    #server {
  -    #    listen 12345;
  -    #    status_zone tcp_server;
  -    #    proxy_pass stream_backend;
  -    #}
  -#}


NAP DoSの設定を含むコンフィグファイルを確認します。server block にて各種 L7Dos の設定を読み込んでいます。

.. code-block:: cmdin

  cat /etc/nginx/conf.d/default.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  upstream server_group {
      zone backend 64k;
      server security-backend1:80;
  }
  
  log_format log_dos ', vs_name_al=$app_protect_dos_vs_name, ip=$remote_addr, tls_fp=$app_protect_dos_tls_fp, outcome=$app_protect_dos_outcome, reason=$app_protect_dos_outcome_reason, ip_tls=$remote_addr:$app_protect_dos_tls_fp, ';
  
  # dos
  server {
      listen 8080;
      keepalive_requests 100000;
      server_name juiceshop;
      ssl_certificate_key conf.d/ssl/nginx-ecc-p256.key;
      ssl_certificate conf.d/ssl/nginx-ecc-p256.pem;
      ssl_session_cache shared:SSL:10m;
      ssl_session_timeout 5m;
      ssl_ciphers AES128-GCM-SHA256;
      ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
      app_protect_dos_security_log_enable on;
      app_protect_dos_security_log "/etc/app_protect_dos/log-default.json" syslog:server=elasticsearch:5261;
      access_log syslog:server=elasticsearch:5561 log_dos;
  
      location / {
          app_protect_dos_enable on;
          app_protect_dos_name "juiceshop";
          app_protect_dos_monitor uri=http://security-backend1:80/ timeout=3;
          app_protect_dos_policy_file "/etc/app_protect_dos/BADOSDefaultPolicy.json";
          proxy_pass http://server_group;
      }
  }


| NAP Dosでは、ログに関する設定をJSONファイルで指定します。
| デフォルトの設定ファイルを利用します。ファイルの内容を確認します。

.. code-block:: cmdin

  cat /etc/app_protect_dos/log-default.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  {
    "filter": {
      "traffic-mitigation-stats": "all",
      "bad-actors": "top 10",
      "attack-signatures": "top 10"
    }
  }


| NAP Dosは、NAP Dosの動作をJSONファイルで指定します。
| 設定ファイルの内容を確認します。

.. code-block:: cmdin

  cat /etc/app_protect_dos/BADOSDefaultPolicy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  {
      "mitigation_mode" : "standard",
      "signatures" : "on",
      "bad_actors" : "on",
      "automation_tools_detection" : "on",
      "tls_fingerprint" : "on"
  }
  
.. code-block:: cmdin

  cat /etc/nginx/conf.d/plus_api.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  
  server {
      listen 8888;
      access_log /var/log/nginx/mng_access.log;
  
      location /api {
          app_protect_dos_api;
      }
  
      location = / {
          rewrite ^(.*)$ https://$host/dashboard-dos.html permanent;
      }
  
      location = /dashboard-dos.html {
          root   /usr/share/nginx/html;
      }
  
  }


プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload


2. 事前準備
----

1. 疎通確認
~~~~

まず初めにサンプルアプリケーションにアクセスできることを確認します。

| バックエンドには ``OWASP Juice Shop`` というアプリケーションが動作しています。
| 正しく接続できることを確認します

.. code-block:: cmdin

  curl -s localhost:8080  | grep title

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  <title>OWASP Juice Shop</title>

2. ELKの確認
~~~~

この通信の結果をELKで取得していることを確認します

``Jump Host`` でブラウザを起動し、 `http://elk:5601 <http://elk:5601>`__ を開いてください

.. NOTE::
   クライアント端末のブラウザより、以下の手順で接続いただくことも可能です

   .. image:: ../module03/media/udf_docker_elk.jpg
       :width: 200

左上メニューを開いてください。

   .. image:: ../module03/media/elk-menu.jpg
       :width: 400

``Discover`` をクリックし、表示された画面の `+ Add filter` の下にすでに登録されている ``app-protect-dos-logs`` を選択してください

   .. image:: ./media/elk-discover-doslogs.png
       :width: 400

| 正しくNAP DoSよりログが転送されている場合、画面のようなグラフが表示されます。
| 画面の内容が最新の状態となっていない場合、画面右上の時間を確認の上、 ``Refresh`` をクリックしてください。

   .. image:: ./media/elk-discover-dos.png
       :width: 400

左上テキストボックスに ``vs_name_al`` と入力し、Enter を押してください。以下のように該当のログが出力されることが確認できます

   .. image:: ./media/elk-discover-dos2.png
       :width: 400


3. NGINX Plus Dashboard の確認
~~~~

NGINX Plus Dashboardで今後ステータスを確認するため、ブラウザでアクセスしておきます

作業を行うホストからブラウザでNGINX Plus Dashboardを開く場合、 ``ubuntu01`` の接続はメニューより ``PLUS  DASHBOARD`` をクリックしてください。
踏み台ホストから接続する場合、ブラウザで `http://10.1.1.7:8888/dashboard-dos.html <http://10.1.1.7:8888/dashboard-dos.html>`__ を開いてください

表示されたオブジェクトをクリックしてください

   - .. image:: ./media/plus-dashboard-dos.png
       :width: 400

   - .. image:: ./media/plus-dashboard-dos2.png
       :width: 400

- ``Name`` が保護対象となる ``app_protect_dos_name`` で指定したオブジェクト名が表示されます
- ``Health`` から ``Learning`` が現在の状態を示します
- ``Protocol`` から ``TLS Fingerprint`` は設定で指定した内容を示します

4. 作業ホストへ接続
~~~~

正常な通信は ``docker_host(10.1.1.5)`` より、攻撃トラフィックは ``ubuntu02(10.1.1.6)`` より実行します

Windows Jump Hostへログインいただくと、SSHClientのショートカットがありますので、そちらをダブルクリックし
``docker_host`` へ接続ください

   - .. image:: ../module01/media/putty_icon.jpg
      :width: 50

   - .. image:: ../module01/media/putty_menu.jpg
      :width: 200

双方のホストで必要なファイルを取得します

.. code-block:: cmdin
   
   sudo su -
   git clone https://github.com/BeF5/f5j-nginx-plus-lab2-security-conf.git

スクリプトに実行権限を付与します

.. code-block:: cmdin

   cd ~/f5j-nginx-plus-lab2-security-conf/l7dos/
   chmod +x *sh

2. Slow HTTPの実施
====

1. ベーストラフィックの実行
----

以下の作業は、 ``docker_host(10.1.1.5)`` にて実行します

| 以下コマンドを実行し、ベースとなる通信を実行します。
| ベースラインを作成するために ``10分`` 程度経過した後次のタスクを実施してください。

.. code-block:: cmdin

  ./good.sh
  # NAP DoSのラボが完了後、Ctrl+C で停止していただきます

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  JUICESHOP HTTP Code:200
  
  JUICESHOP HTTP Code:200
  ...

2. Dashboardの表示
----

1. NGINX Plus Dashboardの表示
~~~~

Dashboardを開くと、トラフィックが転送されている状態が確認できます

   .. image:: ./media/plus-dashboard-dos-good.png
       :width: 400

一定時間トラフィックを正しく転送し学習が完了した場合、 ``Learning`` が ``Ready`` となります

.. NOTE::

  NGINX Plus Dashboard の NAP DoS の Graph 表示は崩れることが多いので参考程度に参照してください

2. ELK ステータスの確認
~~~~

左上メニューを開いてください。

   .. image:: ../module03/media/elk-menu.jpg
       :width: 400

``Dashboard`` をクリックし、 ``AP_DOS: AppProtectDOS`` を開きます

   .. image:: ./media/elk-dashboard-dos.png
       :width: 400

.. NOTE::
   Dashboardの左上テキストボックスに、デフォルトの条件が指定されているので削除してすべての情報が表示されるように変更ください

   .. image:: ./media/elk-dashboard-dos-delete-example.png
       :width: 400

``docker_host(10.1.1.5)`` からのトラフィックを受信し、すべて ``Allow(許可)`` となっていることがわかります

   .. image:: ./media/elk-dashboard-dos-good.png
       :width: 400

グラフの表示項目は以下の内容です

- ``Top talkers`` : 保護対象サービスに対し接続を行うクライアント、及びその各通信制御状況
- ``Client HTTP transactions/s`` : クライアントのHTTP Transaction
- ``HTTP mitigation`` : Mitigationの状況
- ``Server HTTP transactions/s`` : 転送先サーバのHTTP Transaction
- ``Server_stress_level`` : 転送先サーバのストレスレベル
- ``Attack signatures`` : Attack Signatureが生成されたログ
- ``Detected bad actors`` : Bad Actorが検知されたログ 
- ``Geo`` : Geolocationを用いた送信元情報

.. NOTE::
   実際のラボでは、 ``AP_DOS: Top talkers`` に ``127.0.0.1`` 初回に実施した ``Curl`` が含まれますがその結果は除外してご確認ください

3. Slow HTTPの実施
----

以降の作業は、 ``ubuntu02(10.1.1.6)`` で実行します。
以下コマンドを実行し ``Slow HTTP`` を発生させ、状態を確認します。 ``3分`` を目安に攻撃コマンドを ``Ctrl+C`` で停止してください

.. NOTE::
   長時間にわたり攻撃を行った場合、ベースラインのスクリプトも制御対象に含まれ、通信が拒否される場合があります

Docker RUNにてコンテナを起動するため、はじめにコンテナイメージを取得し、その後トラフィックを実行します

.. code-block:: cmdin

  sudo docker run --rm shekyan/slowhttptest:latest -c 50000 -B -g -o my_body_stats -l 600 -i 5 -r 1000 -s 8192 -u http://10.1.1.7:8080/rest/products/search?q=vodka  -x 10 -p 3
  # 本項目の動作を確認後、Ctrl+C で停止してください

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

         slowhttptest version 1.8.3
   - https://github.com/shekyan/slowhttptest -
  test type:                        SLOW BODY
  number of connections:            50000
  URL:                              http://10.1.1.7:8080/rest/products/search?q=vodka
  verb:                             POST
  cookie:
  Content-Length header value:      8192
  follow up data max size:          22
  interval between follow up data:  5 seconds
  connections per seconds:          1000
  probe connection timeout:         3 seconds
  test duration:                    600 seconds
  using proxy:                      no proxy
  
  Tue Jan 24 10:27:04 2023:
  slow HTTP test status on 0th second:
  
  initializing:        0
  pending:             1
  connected:           0
  error:               0
  closed:              0
  service available:   YES

  一定時間経過後、テストが開始します


1. NGINX Plus Dashboardの表示
~~~~

Dashboardを開いてください。正しく表示されない場合ブラウザの更新ボタンをクリックしてください。

   .. image:: ./media/plus-dashboard-dos-slowhttp.png
       :width: 400

Slow HTTP攻撃が開始した後、以下の項目が変化しています

- ``Under Attack`` が ``yes`` と表示される
- Traffic の ``Mitigation/s (Mitigation per sec)`` に現在の状況が表示される
- Traffic の ``Mitigation`` の値が増加する
- Graphの ``Mitigations`` が増加し、赤色で表示される

2. ELK ステータスの確認
~~~~

こちらの表示結果は、攻撃を実行して一定時間立った結果を示しています

NAP DoS で保護対象となっているオブジェクト・グラフが表示されます。
画面下部のグラフを表示する

   .. image:: ./media/elk-dashboard-dos-slowhttp.png
       :width: 400

- ``Client HTTP transactions/s`` 、 ``Server HTTP transactions/s`` の内容を確認すると、予め実行した正常（と想定した通信）によるBaselineと同等の ``Successful tps`` となっていることがわかります。また、攻撃を検知した後、攻撃トラフィックを緩和し適切にサーバに転送していることがわかります
- ``HTTP Mitigation`` の内容を確認すると、攻撃を検知した後、状況に応じて順次通信の内容に応じて様々な緩和策を実施していることがわかります
- ``Attack signatures`` 、 ``Detected bad actors`` のそれぞれに記録されたログが出力されています

Attack signaturesのログが記録されていることがわかります。内容を確認します

   .. image:: ./media/elk-dashboard-dos-slowhttp-sig.png
       :width: 400

``signatures`` 欄に記載されている内容は以下となります。読みやすいように改行しています

.. code-block:: bash
  :linenos:
  :caption: Attack Signature 表示内容サンプル

  (http.hdrorder hashes-to 24) 
  and (http.referer_header_exists eq true) 
  and (http.request.method eq POST) 
  and (http.user_agent contains other-than(IE|Firefox|Opera|Chrome|Safari|curl|grpc)) 
  and (http.referer hashes-to 51) 
  and (http.accept contains text/javascript) 
  and (http.x_forwarded_for_header_exists eq false) 
  and (http.connection_header_exists eq true) 
  and (http.uri_parameters eq 0) 
  and (http.uri_len between 16-31) 
  and (http.content_type_header_exists eq true) 
  and (http.cookie_header_exists eq false) 
  and (http.uri_file hashes-to 31) 
  and (http.headers_count eq 7) 
  and (http.content_length_header_exists eq true) 
  and (http.accept_header_exists eq true) 
  and (http.host_header_exists eq true) 
  and (http.user_agent_header_exists eq true)


Bad Actorのログが記録されていることがわかります。内容を確認します

   .. image:: ./media/elk-dashboard-dos-slowhttp-badactor.png
       :width: 400

.. NOTE::
  ``docker_host(10.1.1.5) が bad actor`` として検知されている場合には、 次のラボで ``docker_host(10.1.1.5) が bad actorとして検知された場合`` を参照してください

4. Slow HTTPの停止
----

攻撃トラフィックが動作中の場合、 ``ubuntu02(10.1.1.6)`` のターミナルで
実行したコマンドを ``Ctrl+C`` で停止してください

2. HTTP Floodの実施
====

1. ベース通信の実施
----

Slow HTTPのベース通信を実行中の場合はこちらのステップを飛ばしてください。
必要に応じて再度ベース通信を、 ``docker_host(10.1.1.5)`` にて実行します

.. NOTE::
  ``docker_host(10.1.1.5) が bad actorとして検知された場合`` 、 ``ubuntu01 (10.1.1.7)`` でNGINXを再起動し、再度ベーストラフィックを学習してください

  .. code-block:: bash

    service nginx restart

.. code-block:: cmdin

  # cd ~/f5j-nginx-plus-lab2-security-conf/l7dos/
  # chmod +x *sh
  ./good.sh
   # NAP DoSのラボが完了後、Ctrl+C で停止していただきます

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  JUICESHOP HTTP Code:200
  
  JUICESHOP HTTP Code:200
  ....

2. Dashboardの表示
----

Slow HTTPの手順と同様に、 ``NGINX Plus Dashbaord`` 、 ``ELK`` の内容が表示されることを確認してください

3. HTTP Floodの実施
----

以降の作業は、 ``ubuntu02(10.1.1.6)`` で実行します。
以下コマンドを実行し ``Slow HTTP`` を発生させ、状態を確認します

.. code-block:: cmdin

  # cd ~/f5j-nginx-plus-lab2-security-conf/l7dos/
  # chmod +x *sh
  ./http1flood.sh

  # 本項目の動作を確認後、Ctrl+C で停止していただきます

Docker RUNにてコンテナを起動するため、はじめにコンテナイメージを取得します。
その後、攻撃に該当する通信が実行されます

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  Benchmarking 10.1.1.7 (be patient)
  Completed 200 requests
  Completed 400 requests
  Completed 600 requests
  Completed 800 requests
  Completed 1000 requests
  Completed 1200 requests
  Completed 1400 requests
  Completed 1600 requests
  Completed 1800 requests
  Completed 2000 requests
  Finished 2000 requests
  
  
  Server Software:        nginx/1.23.2
  Server Hostname:        10.1.1.7
  Server Port:            8080
  
  Document Path:          /rest/products/search?q=vodka
  Document Length:        Variable
  
  Concurrency Level:      100
  Time taken for tests:   4.121 seconds
  Complete requests:      2000
  Failed requests:        0
  Non-2xx responses:      1714
  Total transferred:      877982 bytes
  HTML transferred:       301674 bytes
  Requests per second:    485.27 [#/sec] (mean)
  Time per request:       206.070 [ms] (mean)
  Time per request:       2.061 [ms] (mean, across all concurrent requests)
  Transfer rate:          208.04 [Kbytes/sec] received
  
  Connection Times (ms)
                min  mean[+/-sd] median   max
  Connect:        0    1   0.9      1       7
  Processing:     0  186 458.1      2    1886
  Waiting:        0  185 458.3      2    1886
  Total:          0  187 458.2      3    1890

4. Dashboardの表示
----

1. NGINX Plus Dashboardの表示
~~~~

Dashboardを開いてください。正しく表示されない場合ブラウザの更新ボタンをクリックしてください。

   .. image:: ./media/plus-dashboard-dos-httpflood.png
       :width: 400

HTTP Flood攻撃が開始した後、Slow HTTPと同様に以下の項目が変化しています

- ``Under Attack`` が ``yes`` と表示される
- Traffic の ``Mitigation/s (Mitigation per sec)`` に現在の状況が表示される（サンプルはスクリーンショット取得のタイミングで通信が発生しなかったため 0)
- Traffic の ``Mitigation`` の値が増加する
- Graphの ``Mitigations`` が増加し、赤色で表示される

2. ELK ステータスの確認
~~~~

NAP DoS で保護対象となっているオブジェクト・グラフが表示されます。
画面下部のグラフを表示する

   .. image:: ./media/elk-dashboard-dos-httpflood.png
       :width: 400

HTTP Flood攻撃が開始した後、以下の項目が変化しています

- ``Client HTTP transactions/s`` 、 ``Server HTTP transactions/s`` の内容を確認すると、予め実行した正常（と想定した通信）によるBaselineと同等の ``Successful tps`` となっていることがわかります。また、攻撃を検知した後、攻撃トラフィックを緩和し適切にサーバに転送していることがわかります
- Slow HTTPと異なりHTTP Flood の場合には、通信の内容がほぼ正常な通信のため瞬間的にサーバサイドへ転送されますがその後緩和されていることがわかります
- ``HTTP Mitigation`` の内容を確認すると、攻撃を検知した後、状況に応じて順次通信の内容に応じて様々な緩和策を実施していることがわかります
- ``Attack signatures`` 、 ``Detected bad actors`` のそれぞれに記録されたログが出力されています

Attack Signatureのログが記録されていることがわかります。内容を確認します

   .. image:: ./media/elk-dashboard-dos-httpflood-sig.png
       :width: 400

``signatures`` 欄に記載されている内容は以下となります。読みやすいように改行しています

.. code-block:: bash
  :linenos:
  :caption: Attack Signature 表示内容サンプル

  (http.x_forwarded_for_header_exists eq false) 
  and (http.cookie_header_exists eq false) 
  and (http.uri_file hashes-to 31) 
  and (http.user_agent contains other-than(IE|Firefox|Opera|Chrome|Safari|curl|grpc)) 
  and (http.hdrorder hashes-to 50) 
  and (http.request.method eq GET) 
  and (http.headers_count eq 3) 
  and (http.uri_parameters eq 0) 
  and (http.accept contains other-than(application|audio|message|text|image|multipart)) 
  and (http.uri_len between 16-31) 
  and (http.host_header_exists eq true) 
  and (http.accept_header_exists eq true) 
  and (http.user_agent_header_exists eq true)

Bad Actorのログが記録されていることがわかります。内容を確認します

   .. image:: ./media/elk-dashboard-dos-httpflood-badactor.png
       :width: 400
