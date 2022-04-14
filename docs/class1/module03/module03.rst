NGINX App Protect WAF (NAP WAF)
#######

NAP WAF について
====

NAP WAFは、ワールドワイドで実績豊富なF5製WAFの機能を移植した、NGINX Plusの動的モジュールで実現するWAFです。

   .. image:: ./media/nap-waf.jpg
       :width: 400

NAP WAFはNGINXの動的モジュールであるという特徴から、GatewayからIngress Controller、更にコンテナとして柔軟なデプロイが可能です。
アプリケーションの性質によりセキュリティ機能の実装方法が異なると思いますが、NAP WAFのはその柔軟な適応力から最適な形でWAFを実現することができます。

   .. image:: ./media/nap-waf-structure.jpg
       :width: 400

以下項目では主要な設定について、それぞれの設定と動作を確認します。
設定の詳細は、 `こちら <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/>`__ の内容を参照してください。

1. シンプルなWAFの設定
====

1. 設定
----

WAFを設定します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   cp ~/back-to-basic_plus-security/waf/waf-l1_demo.conf default.conf
   cp ~/back-to-basic_plus-security/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/back-to-basic_plus-security/waf/waf-l1_custom_policy.json custom_policy.json


WAFを設定を確認します

設定ファイルを確認します。 ``listen 80`` の server block にて各種WAFの設定を読み込んでいます。

.. code-block:: cmdin

  cat default.conf

.. code-block:: bash
  :caption: 実行結果サンプル

  upstream server_group {
      zone backend 64k;
      server security-backend1:80;
  }
  # waf
  server {
      listen 80;
      app_protect_enable on;
      app_protect_security_log_enable on;
      app_protect_security_log "/etc/nginx/conf.d/custom_log_format.json" syslog:server=elasticsearch:5144;
  
      location / {
          app_protect_policy_file "/etc/nginx/conf.d/custom_policy.json";
  
          proxy_pass http://server_group;
      }
  }
  # no waf
  server {
      listen 81;
      location / {
          proxy_pass http://server_group;
      }
  }


NAP WAFでは、ログフォーマットをJSONファイルで指定します。
設定ファイルの内容を確認します。

.. code-block:: cmdin

  cat custom_log_format.json

.. code-block:: bash
  :caption: 実行結果サンプル


  {
      "filter": {
          "request_type": "all"
      },
      "content": {
          "format": "default",
          "max_request_size": "any",
          "max_message_size": "10k"
      }
  }


NAP WAFでは、WAFののセキュリティポリシーをJSONファイルで指定します。
設定ファイルの内容を確認します。

.. code-block:: cmdin

  cat custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル


  {
      "policy":
      {
          "name": "policy-acceptall",
          "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
          "applicationLanguage": "utf-8",
          "enforcementMode": "transparent"
      }
  }
  

このサンプルでは、まずWAF設定が正しく設定されることを確認しています。
``enforcementMode`` で ``transparent`` と指定しているため、通信のBlockは行われません。

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload


2. 動作確認
----

まず初めにサンプルアプリケーションにアクセスできることを確認します。

バックエンドには ``OWASP Juice Shop`` というアプリケーションが動作しています。
正しく接続できることを確認します

.. code-block:: cmdin

  curl -s localhost  | grep title

.. code-block:: bash
  :caption: 実行結果サンプル

  <title>OWASP Juice Shop</title>


この通信の結果をELKで取得していることを確認します

ラボの 構成全体の画面を開き、 ``ELK`` を開いてください

   .. image:: ./media/udf_docker_elk.jpg
       :width: 200

左上メニューを開き ``Discover`` をクリックしてください

   .. image:: ./media/elk-menu.jpg
       :width: 400

   .. image:: ./media/elk-menu2.jpg
       :width: 200

表示された画面の `+ Add filter` の下にすでに登録されている ``waf-logs-*`` を選択肢てください

   .. image:: ./media/elk-discover-waf.jpg
       :width: 400

正しくNAP WAFよりログが転送されている場合、画面のようなグラフが表示されます。
画面の内容が最新の状態となっていない場合、画面右上の時間を確認の上、 ``Refresh`` をクリックしてください。

   .. image:: ./media/elk-discover-waf2.jpg
       :width: 400

表示されたログの詳細を一つ確認してみましょう。
当該のログの左側 ``∨`` をクリックすると詳細が表示されます。参考に内容を確認すると ``bot_signature_name`` に ``curl`` と表示されていることがわかります。

   .. image:: ./media/elk-l1-discover.jpg
       :width: 400

通信は確認した通り許可されておりますが、Curlコマンドを利用した通信が到達していることが確認できます。


2. 通信のブロック(enforcementMode)
====


1. 設定
----

通信のブロックを行うため設定を変更します。
``Default Policy`` の設定・動作の詳細については、 `こちら Basic Configuration and the Default Policy <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/#basic-configuration-and-the-default-policy>`__ を参照してください

それでは、WAFのセキュリティポリシーのみ変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/back-to-basic_plus-security/waf/waf-l1_demo.conf default.conf
   # cp ~/back-to-basic_plus-security/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/back-to-basic_plus-security/waf/waf-l2_custom_policy.jsonn custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/back-to-basic_plus-security/waf/waf-l1_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル

   --- /root/back-to-basic_plus-security/waf/waf-l1_custom_policy.json     2022-04-14 23:27:19.383236359 +0900
   +++ custom_policy.json  2022-04-14 23:21:06.978541812 +0900
   @@ -4,6 +4,6 @@
            "name": "policy-acceptall",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "transparent"
   +        "enforcementMode": "blocking"
        }
    }

``enforcementMode`` で ``blocking`` と指定されていることがわかります。
この設定により通信をブロックすることが可能です。


プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

2. 動作確認
----

クロスサイトスクリプティング(XSS)に該当する通信を発生させます。以下のCurlコマンドを実行し、結果を確認します。

.. code-block:: cmdin

  curl -s "localhost/?a=<script>"

.. code-block:: bash
  :caption: 実行結果サンプル

  <html>
      <head>
          <title>Request Rejected</title>
      </head>
      <body>The requested URL was rejected. Please consult with your administrator.<br><br>
          Your support ID is: 935833362169160317<br><br>
          <a href='javascript:history.back();'>[Go Back]</a>
      </body>
  </html>

先程確認したようにバックエンドのアプリケーションは表示されず、 ``Request Rejected`` の文字とともにHTMLが応答されていることがわかります。
この実行サンプルでは表示を確認するためテキストを一部整形しております。皆様の環境では改行がなく結果が表示されていると思います


それではログの情報を確認します。

``ELK`` > ``Discover`` > ``waf-logs-*`` を開き、表示された結果の ``∨`` をクリックし、詳細を表示してください。

   .. image:: ./media/elk-l2-discover.jpg
       :width: 400

この項目では、ELKのGUIでどの様に表示されるか確認します。

   .. image:: ./media/elk-l2-discover-detail.jpg
       :width: 400

主要な項目について確認します

- 中段に表示されている ``support_id`` をまず最初に説明します。先程のBlock Pageを確認してください。こちらに表示されている `Support ID` と一致していることが確認できます。この様にNAP WAFでは、セキュリティログを一意に特定する値として `Support ID` が存在します。攻撃が拒否された場合には、Block された画面(HTML)により該当のログを特定することが可能です
- ``bot_category`` は `HTTP Library` 、 ``bot_signature_name`` は `curl` となっています
- ``client_class`` は Bot SIgnature によるClassというカテゴリを示しており、 `Untrusted Bot` となっています
- ``outcome`` は 処理の結果を示しており、 ``REJECETD(拒否)`` となることがわかります
- ``sig_`` から始まる項目が、 `signature` に関する情報を示しております。各項目は、Signature ID、Signature Name、Signature Set Name となっています
- ``voilation_`` から始まる項目が、 `VIOLATION` に関する情報を示しております。通信がどのような脅威に該当するのか確認できます
- VIOLATIONの中で ``violation_rating`` があり、これはその重要度(危険度)を示しています。NAP WAFのDefault Policyでは、検知した通信で複数の問題が確認され、その結果判定されたRatingが一定の値より高い場合に通信が拒否される設定となっています。

``JSON`` 形式の表示は、該当ログの ``JSON`` タブをクリックしてください

   .. image:: ./media/elk-l2-discover-json.jpg
       :width: 400

次にELKが提供するもう一つの機能である、 ``Dashboard`` を確認します

``ELK`` > ``Dashboard`` を開き ``Overview`` をクリックしてください。
この画面はWAFのステータスを俯瞰する画面となります。

   .. image:: ./media/elk-l2-dashboard-select-overview.jpg
       :width: 400

通信量が少ないため内容は大変シンプルとなっております。

   .. image:: ./media/elk-l2-dashboard-overview.jpg
       :width: 400

こちらの内容から横断的に通信の状況を把握することができるようになっています。
リクエストの処理結果の割合や、クライアントIPアドレスの分布、アクセス先のURL、検知したSignatureや、Violation等の割当を知ることができます。
この画面を見ることにより、今の検知状況や頻繁に発生・検知している攻撃などを俯瞰的に知ることができます。


``ELK`` > ``Dashboard`` を開き ``False Positives`` をクリックしてください。

   .. image:: ./media/elk-l2-dashboard-select-falsepositive.jpg
       :width: 400

Overviewと同様に結果はシンプルです。

   .. image:: ./media/elk-l2-dashboard-falsepositive.jpg
       :width: 400

以下のグラフは、ある瞬間に特定のシグネチャや違反にヒットしたユニークなIPの総数を表示します。
これらのグラフでスパイクが発生している場合、多くのクライアントが同じルールをトリガーしていることを意味し、特定のクライアントに依存しない検出結果、
つまり ``誤検知の可能性の高い通信`` を見つけることを目的としています。
誤検知と判定された場合には、対象Signatureを除外設定にするなどの対処をセキュリティポリシーに対して実施することとなります。


参考の情報ですが、curlコマンドの `?a=<script>`` を `?a='or+1=1--` などの文字列に入れ替えると、SQL Injectionのブロックを見ることができますのでご確認ください。

a
==================================================================
1. WAFの設定をデプロイ
====

1. 設定
----
2. 動作確認
----


WAFを設定します

.. code-block:: cmdin

  cat default.conf

.. code-block:: bash
  :caption: 実行結果サンプル


プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload




.. code-block:: cmdin

.. code-block:: bash
  :caption: 実行結果サンプル

b
==================================================================
