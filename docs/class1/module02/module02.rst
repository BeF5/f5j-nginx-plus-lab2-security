NGINX LAB
#######

NGINX Plus のインストール test
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

ラボの実施（作成中）
====

必要なパッケージの取得

.. code-block:: cmdin
   
   git clone https://github.com/hiropo20/back-to-basic_plus-security.git


.. code-block:: cmdin
 
  kubectl get pod

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル

  NAME                       READY   STATUS    RESTARTS  AGE
  coffee-7c86d7d67c-wjxss    1/1     Running   0         1m
  coffee-7c86d7d67c-8jm9z    1/1     Running   0         1m
  tea-5c457db9-dc4cs         1/1     Running   0         1m