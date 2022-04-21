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
   