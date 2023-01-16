NGINX LAB
#######

NGINX Plus のインストール
====

1. NGINX Plus、アドオンモジュールのインストール (15min)
----

こちらの手順「 `NGINX Plusのインストール (15min) <https://f5j-nginx-plus-lab1.readthedocs.io/en/latest/class1/module2/module2.html#nginx-plus-15min>`__」を参考に、NGINX Plus、アドオンモジュールをインストールをインストールしてください

手順の内容に加えて、必要となるモジュールをインストールしてください

.. code-block:: cmdin

   sudo apt-get install nginx-plus-module-njs


インストールしたパッケージの情報を確認いただけます


.. code-block:: cmdin

   # dpkg-query -l | grep njs
   
   ii  nginx-plus-module-njs              25+0.7.0-1~focal                      amd64        NGINX Plus njs dynamic modules

ラボの実施
====

1. 必要なパッケージの取得
----

.. code-block:: cmdin
   
   sudo su
   cd ~/
   git clone https://github.com/BeF5/f5j-nginx-plus-lab2-security-conf.git

2. インストールしたNGINX Plusに必要な設定を追加
----

.. code-block:: cmdin
   
   # sudo su
   mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-bak
   cat ~/f5j-nginx-plus-lab2-security-conf/base/loadmodules.conf /etc/nginx/nginx.conf-bak > /etc/nginx/nginx.conf 

設定内容を確認します

.. code-block:: cmdin
   
   head -7  /etc/nginx/nginx.conf

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   # for NAP WAF
   load_module modules/ngx_http_app_protect_module.so;
   # for NAP DoS
   load_module modules/ngx_http_app_protect_dos_module.so;
   # for NJS
   load_module modules/ngx_http_js_module.so;
   load_module modules/ngx_stream_js_module.so;

3. NGINX の起動

.. code-block:: cmdin

   service nginx stop
   service nginx start

対象となるプロセスが動作していることを確認します

.. code-block:: cmdin

   ps -ef | grep -e nginx

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:
  :emphasize-lines: 1-7

   nginx       8408       1  0 08:18 ?        00:00:00 /bin/sh -c usr/share/ts/bin/bd-socket-plugin tmm_count 4 proc_cpuinfo_cpu_mhz 2000000 total_xml_memory 307200000 total_umu_max_size 3129344 sys_max_account_id 1024 no_static_config 2>&1 >> /var/log/app_protect/bd-socket-plugin.log
   nginx       8410    8408  1 08:18 ?        00:00:01 usr/share/ts/bin/bd-socket-plugin tmm_count 4 proc_cpuinfo_cpu_mhz 2000000 total_xml_memory 307200000 total_umu_max_size 3129344 sys_max_account_id 1024 no_static_config
   nginx       8412       1  0 08:18 ?        00:00:00 /bin/sh -c LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/rpm/lib64; export LD_LIBRARY_PATH; /usr/bin/admd -d --log info 2>&1 > /var/log/adm/admd.log
   nginx       8420    8412  0 08:18 ?        00:00:00 /usr/bin/admd -d --log info
   root        8452       1  0 08:18 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
   nginx       8453    8452  0 08:18 ?        00:00:00 nginx: worker process
   nginx       8454    8452  0 08:18 ?        00:00:00 nginx: worker process
   root        8477    8366  0 08:21 pts/0    00:00:00 grep --color=auto -e nginx

- ``1-2行目`` が NGINX App Protect WAFのプロセスです
- ``3-4行目`` が NGINX App Protect DoSのプロセスです
- ``5行目`` が NGINXのマスタープロセス、 ``6-7行目`` がワーカープロセスです