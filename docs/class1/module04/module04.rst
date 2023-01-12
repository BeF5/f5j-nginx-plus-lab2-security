NGINX App Protect Dos (NAP Dos)
#######

NAP Dos について
====

NAP Dosは、ワールドワイドで実績豊富なF5製WAFのL7DoS機能を移植した、NGINX Plusの動的モジュールで実現するWAFです。

NAP DosはNGINXの動的モジュールであるという特徴から、GatewayからIngress Controller、更にコンテナとして柔軟なデプロイが可能です。

..
   .. image:: ./media/nap-waf-structure.jpg
       :width: 400


1. NAP Dos の設定
====

1. 設定
----

NAP Dosを設定します

.. code-block:: cmdin

   # sudo su
   cd /etc/nginx/conf.d
   cp ~/f5j-nginx-plus-lab2-security-conf/l7dos/l7dos-l1_demo.conf default.conf
   cp ~/f5j-nginx-plus-lab2-security-conf/ssl/* ssl/


設定ファイルを確認します。 server block にて各種 L7Dos の設定を読み込んでいます。

.. code-block:: cmdin

  cat default.conf

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
      set $loggable '0';
      access_log syslog:server=elasticsearch:5561 log_dos if=$loggable;
  
      #app_protect_security_log "/etc/nginx/conf.d/custom_log_format.json" syslog:server=elasticsearch:5144;
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
  

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload


2. 動作確認
----

まず初めにサンプルアプリケーションにアクセスできることを確認します。

| バックエンドには ``OWASP Juice Shop`` というアプリケーションが動作しています。
| 正しく接続できることを確認します

.. code-block:: cmdin

  curl -s localhost  | grep title

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  <title>OWASP Juice Shop</title>


この通信の結果をELKで取得していることを確認します

``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください

.. NOTE::
   クライアント端末のブラウザより、以下の手順で接続いただくことも可能です

   .. image:: ../module03/media/udf_docker_elk.jpg
       :width: 200

左上メニューを開いてください。

   .. image:: ../module03/media/elk-menu.jpg
       :width: 400

``Discover`` をクリックし、表示された画面の `+ Add filter` の下にすでに登録されている ``app-protect-dos-logs`` を選択してください

   .. image:: ../module03/media/elk-discover-waflogs.jpg
       :width: 400

| 正しくNAP WAFよりログが転送されている場合、画面のようなグラフが表示されます。
| 画面の内容が最新の状態となっていない場合、画面右上の時間を確認の上、 ``Refresh`` をクリックしてください。

..
   .. image:: ./media/elk-discover-waf2.jpg
       :width: 400

| 表示されたログの詳細を一つ確認してみましょう。
| 当該のログの左側 ``>`` をクリックすると詳細が表示されます。

..
   .. image:: ./media/elk-l1-discover.jpg
       :width: 400



2. ベース通信の実施
====

1. 作業ホストへ接続
----

移行の作業は、 ``docker_host`` より実行します

Windows Jump Hostへログインいただくと、SSHClientのショートカットがありますので、そちらをダブルクリックし
``docker_host`` へ接続ください

   - .. image:: ../module01/media/putty_icon.jpg
      :width: 50

   - .. image:: ../module01/media/putty_menu.jpg
      :width: 200


2. ベーストラフィックの実行
----

| 以下コマンドを実行し、ベースとなる通信を実行します。
| ベースラインを作成するために10分程度経過した後次のタスクを実施してください。

スクリプトに実行権限を付与します

.. code-block:: cmdin

   cd ~/f5j-nginx-plus-lab2-security-conf/l7dos/
   chmod +x *sh

コマンドを実行します

.. code-block:: cmdin

   ./good.sh
   # ラボ終了時、Ctrl+C で停止してください

3. HTTP Floodの実施
====

1. コマンドの実行
----

移行の作業は、 新たにターミナルを開き ``docker_host`` へ接続します。
以下コマンドを実行し ``HTTP Flood`` を発生させ、状態を確認します

.. code-block:: cmdin

   ## cd ~/f5j-nginx-plus-lab2-security-conf/l7dos/
   ./http1flood.sh

   # 本項目の動作を確認後、Ctrl+C で停止してください

2. NGINX Plus Dashboard ステータスの確認
----

3. ELK ステータスの確認
----

4. Slow HTTPの実施
====

1. コマンドの実行
----

移行の作業は、 新たにターミナルを開き ``docker_host`` へ接続します。
以下コマンドを実行し ``Slow HTTP`` を発生させ、状態を確認します

.. code-block:: cmdin

  docker run --rm shekyan/slowhttptest:latest -c 50000 -B -g -o my_body_stats -l 600 -i 5 -r 1000 -s 8192 -u http://10.1.1.7:8080/rest/products/search?q=vodka  -x 10 -p 3
  # 本項目の動作を確認後、Ctrl+C で停止してください

2. NGINX Plus Dashboard ステータスの確認
----

3. ELK ステータスの確認
----