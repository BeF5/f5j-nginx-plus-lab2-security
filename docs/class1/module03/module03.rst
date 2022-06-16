NGINX App Protect WAF (NAP WAF)
#######

NAP WAF について
====

NAP WAFは、ワールドワイドで実績豊富なF5製WAFの機能を移植した、NGINX Plusの動的モジュールで実現するWAFです。

   .. image:: ./media/nap-waf.jpg
       :width: 400

| NAP WAFはNGINXの動的モジュールであるという特徴から、GatewayからIngress Controller、更にコンテナとして柔軟なデプロイが可能です。
| アプリケーションの性質によりセキュリティ機能の実装方法が異なると思いますが、NAP WAFのはその柔軟な適応力から最適な形でWAFを実現することができます。

   .. image:: ./media/nap-waf-structure.jpg
       :width: 400

| 以下項目では主要な設定について、それぞれの設定と動作を確認します。
| 設定の詳細は、 `こちら <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/>`__ の内容を参照してください。

1. シンプルなWAFの設定
====

1. 設定
----

WAFを設定します

.. code-block:: cmdin

   # sudo su
   cd /etc/nginx/conf.d
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_policy.json custom_policy.json


WAFの設定を確認します

設定ファイルを確認します。 ``listen 80`` の server block にて各種WAFの設定を読み込んでいます。

.. code-block:: cmdin

  cat default.conf

.. code-block:: bash
  :linenos:
  :caption: 実行結果サンプル
  :emphasize-lines: 7-10, 13

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


| NAP WAFでは、ログフォーマットをJSONファイルで指定します。
| 設定ファイルの内容を確認します。

.. code-block:: cmdin

  cat custom_log_format.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:


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


| NAP WAFでは、WAFののセキュリティポリシーをJSONファイルで指定します。
| 設定ファイルの内容を確認します。

.. code-block:: cmdin

  cat custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:
  :emphasize-lines: 7

  {
      "policy":
      {
          "name": "policy-acceptall",
          "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
          "applicationLanguage": "utf-8",
          "enforcementMode": "transparent"
      }
  }
  

| このサンプルでは、まずWAF設定が正しく設定されることを確認しています。
| ``enforcementMode`` で ``transparent`` と指定しているため、通信のBlockは行われません。

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

   .. image:: ./media/udf_docker_elk.jpg
       :width: 200

左上メニューを開いてください。

   .. image:: ./media/elk-menu.jpg
       :width: 400

``Discover`` をクリックし、表示された画面の `+ Add filter` の下にすでに登録されている ``waf-logs-*`` を選択してください

   .. image:: ./media/elk-discover-waflogs.jpg
       :width: 400

| 正しくNAP WAFよりログが転送されている場合、画面のようなグラフが表示されます。
| 画面の内容が最新の状態となっていない場合、画面右上の時間を確認の上、 ``Refresh`` をクリックしてください。

   .. image:: ./media/elk-discover-waf2.jpg
       :width: 400

| 表示されたログの詳細を一つ確認してみましょう。
| 当該のログの左側 ``>`` をクリックすると詳細が表示されます。参考に内容を確認すると ``bot_signature_name`` に ``curl`` と表示されていることがわかります。

   .. image:: ./media/elk-l1-discover.jpg
       :width: 400

通信は確認した通り許可されておりますが、Curlコマンドを利用した通信が到達していることが確認できます。


2. 通信のブロック
====


1. 設定
----

通信のブロックを行うため設定を変更します。
``Default Policy`` の設定・動作の詳細については、 `こちら Basic Configuration and the Default Policy <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/#basic-configuration-and-the-default-policy>`__ を参照してください

それでは、WAFのセキュリティポリシーのみ変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_policy.json     2022-04-14 23:27:19.383236359 +0900
   +++ custom_policy.json  2022-04-14 23:21:06.978541812 +0900
   @@ -1,9 +1,9 @@
    {
        "policy":
        {
   -        "name": "acceptall",
   +        "name": "blocking",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "transparent"
   +        "enforcementMode": "blocking"
        }
    }

| ``enforcementMode`` で ``blocking`` と指定されていることがわかります。
| この設定により通信をブロックすることが可能です。

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
  :linenos:

  <html>
      <head>
          <title>Request Rejected</title>
      </head>
      <body>The requested URL was rejected. Please consult with your administrator.<br><br>
          Your support ID is: 935833362169160317<br><br>
          <a href='javascript:history.back();'>[Go Back]</a>
      </body>
  </html>

| 先程確認したようにバックエンドのアプリケーションは表示されず、 ``Request Rejected`` の文字とともにHTMLが応答されていることがわかります。
| この実行サンプルでは表示を確認するためテキストを一部整形しております。皆様の環境では改行がなく結果が表示されていると思います


| それではログの情報を確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください
| ``ELK`` > ``Discover`` > ``waf-logs-*`` を開き、表示された結果の ``>`` をクリックし、詳細を表示してください。

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

| ``ELK`` > ``Dashboard`` を開き ``Overview`` をクリックしてください。
| この画面はWAFのステータスを俯瞰する画面となります。

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

| 以下のグラフは、ある瞬間に特定のシグネチャや違反にヒットしたユニークなIPの総数を表示します。
| これらのグラフでスパイクが発生している場合、多くのクライアントが同じルールをトリガーしていることを意味し、特定のクライアントに依存しない検出結果、
| つまり ``誤検知の可能性の高い通信`` を見つけることを目的としています。
| 誤検知と判定された場合には、対象Signatureを除外設定にするなどの対処をセキュリティポリシーに対して実施することとなります。


3. 特定Signatureの除外設定
====

1. シナリオ
----

この項目のシナリオは以下となります。

1. バックエンドのアプリケーションの動作確認のため、 ``SQL Injection`` がどの様に制御されるか確認する
2. アプリケーションの前段にWAFが配置されているため、一次的にWAFで ``SQL Injection`` を検知しないように変更する

本来こういったシナリオは多くないかもしれません。WAFのオペレーションの流れとしてご認識ください。

2. ブラウザから攻撃の実施
----

セキュリティポリシーの設定は `2. 通信のブロック <https://f5j-nginx-plus-lab2-security.readthedocs.io/en/latest/class1/module03/module03.html#id3>`__ のポリシーを利用します

``Jump Host`` より ``Owasp Juice Shop`` にアクセスします。ブラウザを起動し、 ``http://juice-shop`` を開いてください

| 画面右上 ``Account`` > ``Login`` をクリックします。
| すでに別のアカウントでログインしている場合、一度ログアウトをしてからこの作業を行ってください。

   .. image:: ./media/owasp-js-login-injection.jpg
       :width: 400

| 表示された画面に以下の内容を入力します。
| 詳細を確認される場合、開発者ツールの ``Network`` を表示してください。

+----------+------------------------------+
| Email    | ``' or 1=1--``               |
+----------+------------------------------+
| Password | ``1`` (どの文字でも良いです) |
+----------+------------------------------+

| 上記のログイン内容は、 ``SQL Injection`` を意図した入力となります。WAFが有効でない場合、認証が回避されログインが完了してしまいます。
| (余裕がある方は、WAFのポリシーを transparent に変更し、動作を確認してみてください)

Webページ側で期待した応答と異なるため、 ``[object Object]`` というエラーとなり、ブロックされています。

   .. image:: ./media/owasp-js-login-injection-block.jpg
       :width: 400

| 開発者ツールの ``Network`` を開き、検索ボックスに ``location`` を入力してください。
| 下のリクエストの ``login`` を選択し、 ``Response`` を確認すると、NAP WAFが応答した情報が返されていることがわかります。

それではログの情報を確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください

| ``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。
| 画面上部の検索窓に ``SQL`` と入力し ``Enter`` を押してください

表示された結果の ``>`` をクリックし、詳細を表示してください。

   .. image:: ./media/elk-overview-sqli-signatureids.jpg
       :width: 400

内容を確認すると、 ``SQL Injection`` として検知され通信がブロックされたことがわかります。

次に画面で ``sig_ids`` を検索し、右側に表示されている内容をご確認ください

.. code-block:: bash
  :caption: 表示結果例
  :linenos:

  200002147, 200002419, 200002883, 200002476, 200015112

後ほどURL Pathの情報も利用いたしますので ``uri`` の欄に表示される内容をご確認ください

.. code-block:: bash
  :caption: 表示結果例
  :linenos:

  /rest/user/login

3. 設定
----

SQL Injection を許可する設定を行います。
WAFのセキュリティポリシーを変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l3_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-14 23:27:46.608110394 +0900
   +++ custom_policy.json  2022-04-19 16:53:25.046672699 +0900
   @@ -1,9 +1,31 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "disable-sqli-signatures",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +       "signatures": [
   +           {
   +                "signatureId": 200002147,
   +                "enabled": false
   +           },
   +           {
   +                "signatureId": 200002419,
   +                "enabled": false
   +           },
   +           {
   +                "signatureId": 200002883,
   +                "enabled": false
   +           },
   +           {
   +                "signatureId": 200002476,
   +                "enabled": false
   +           },
   +           {
   +                "signatureId": 200015112,
   +                "enabled": false
   +           }
   +       ]
        }
    }


``signatures`` に 先程確認した各Signatureが、 ``"enabled": false`` と指定されていることがわかります。
表示されている ``signatureID`` が先程画面で確認した内容と一致していることを確認してください。

この設定により通信をブロックすることが可能です。

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

4. 動作確認
----

``Jump Host`` より ``Owasp Juice Shop`` にアクセスします。ブラウザを起動し、 ``http://juice-shop`` を開いてください

| 画面右上 ``Account`` > ``Login`` をクリックします。
| すでに別のアカウントでログインしている場合、一度ログアウトをしてからこの作業を行ってください。

   .. image:: ./media/owasp-js-login-injection.jpg
       :width: 400

表示された画面に以下の内容を入力します。

+----------+------------------------------+
| Email    | ``' or 1=1--``               |
+----------+------------------------------+
| Password | ``1`` (どの文字でも良いです) |
+----------+------------------------------+

ログインが完了し、TOPページが表示されました。画面右上 ``Account`` をクリックすると、 ``SQL Injection`` により認証を回避し、 ``admin@...`` でログインできていることが確認できます。
（SQL Injection の攻撃が成功した状態です）

   .. image:: ./media/owasp-js-login-injection-successed.jpg
       :width: 400

.. NOTE::
    | このサーバはセキュリティハックのトレーニング用のアプリケーションとなります。
    | 様々な操作が、セキュリティに関する操作に該当する場合があり、POP Upで得点を獲得した
    | 情報が表示されますが無視してください

    .. image:: ./media/owasp-juiceshop-js-popup.jpg
       :width: 400


それではログの情報を確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください

``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。
画面上部の検索窓に ``SQL`` と入力し ``Enter`` を押してください

   .. image:: ./media/elk-discover-waflogs.jpg
       :width: 400

ログを検索いただくと、直前に接続した内容は ``SQL Injection`` としての表示が無いかと思います。

   .. image:: ./media/elk-overview-exclude-sqli.jpg
       :width: 400

| 次に先程確認したURLを検索箇所に入力して状態を確認します。
| 画面上部の ``+Add filter`` をクリックし、画面に表示される項目に以下の内容を入力し　``Save`` をクリックしてください。

+---------+----------------------+
|Field    | ``uri``              |
+---------+----------------------+
|Operator | ``is``               |
+---------+----------------------+
|Vsalue   | ``/rest/user/login`` |
+---------+----------------------+

表示された結果の ``>`` をクリックし、詳細を表示してください。

   .. image:: ./media/elk-overview-exclude-sqli2.jpg
       :width: 400

表示結果を確認すると、uri として該当するログであることがわかります。また ``outcome`` が ``PASSED`` となり許可されていること、 ``sig_ids`` は ``該当なし(N/A)`` であることが確認できます。

4. Custom Blocking Page
====

| 先程SQL InjectionをWAFでBlockしたさい、SPA(Sinagle Page Application) である ``OWASP Juice Shop`` では内容の推察ができない内容となっていました。
| Blockの際に表示される情報を変更し、利用者にとってわかりやすくなるように設定をします

1. 設定
----

Custom Block Pageの設定を行います。
WAFのセキュリティポリシーを変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l4_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-20 10:00:50.107946293 +0900
   +++ custom_policy.json  2022-04-20 14:07:56.299902065 +0900
   @@ -1,9 +1,17 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "custom-blockingpage",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +        "response-pages": [
   +            {
   +                "responseContent": "Attack is detected ID: <%TS.request.ID()%>",
   +                "responseHeader": "HTTP/1.1 401 UnauthorizedK\r\nCache-Control: no-cache\r\nPragma: no-cache\r\nConnection: close",
   +                "responseActionType": "custom",
   +                "responsePageType": "default"
   +            }
   +        ]
        }
    }


| ``response-pages`` に Custom Pageを指定しています。
| ``responseContent`` Block時の応答を HTML で記述することが可能です。
| 今回のアプリケーションに合わせ、 ``responseContent`` 、 ``responseHeader`` を指定しています。

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

2. 動作確認
----

``Jump Host`` より ``Owasp Juice Shop`` にアクセスします。ブラウザを起動し、 ``http://juice-shop`` を開いてください

| 画面右上 ``Account`` > ``Login`` をクリックします。
| すでに別のアカウントでログインしている場合、一度ログアウトをしてからこの作業を行ってください。

   .. image:: ./media/owasp-js-login-injection.jpg
       :width: 400

表示された画面に以下の内容を入力します。

+----------+------------------------------+
| Email    | ``' or 1=1--``               |
+----------+------------------------------+
| Password | ``1`` (どの文字でも良いです) |
+----------+------------------------------+

| エラーでブロックされるため、ログインは行われません。
| 画面には赤文字で ``Attack is detected ID: **support ID**`` が表示されており、セキュリティポリシーで指定した内容となっていることが確認できます。

   .. image:: ./media/owasp-js-login-custom-blockpage.jpg
       :width: 400

ログを確認すると、 ``SQL Injeciton`` の動作確認と同様の内容が確認できます。

| ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。
| ``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。
| 画面上部の検索窓に ``SQL`` と入力し ``Enter`` を押してください。
| ``SQL Injection`` として検知され通信がブロックされたことがわかります。

   .. image:: ./media/elk-overview-sqli-custom-blockpage.jpg
       :width: 400


5. Sensitive Parameter
====

アプリケーションとの通信で重要なデータを取り扱う場合があります。
それらの情報を扱うパラメータを ``sensitive parameter`` に指定することでログやリクエストで情報をマスクすることが可能です

1. 設定
----

sensitive parameterの設定を行います。
WAFのセキュリティポリシーを変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l5_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-20 10:00:50.107946293 +0900
   +++ custom_policy.json  2022-04-20 17:24:28.367618648 +0900
   @@ -1,9 +1,14 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "sensitive-parameter",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +        "sensitive-parameters": [
   +            {
   +                "name": "mypass"
   +            }
   +        ]
        }
    }


``sensitive-parameter`` として ``mypass`` を指定しています。指定のパラメータについては情報がマスクされます

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

2. 動作確認
----

Curl コマンドを使ってリクエストを送信します。 ``mypass`` というパラメータに対し、 ``dummy`` という値を指定しています。

.. code-block:: cmdin

  curl -s "localhost/?mypass=dummy" | grep title

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  <title>OWASP Juice Shop</title>

通信はエラーなく終了しました。

| ログを確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。
| ``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。

   .. image:: ./media/elk-discover-waflogs.jpg
       :width: 400

画面上部の検索窓に ``mypass`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-mypass.jpg
       :width: 400

該当の通信が表示されています。通信の詳細を確認すると ``request`` でURIの情報が確認でき、
パラメータとして指定した ``mypass`` の値がマスクされていることがわかります。

この様に、sensitive-parameter を利用することで、対象の値をマスクすることが可能です。


6. 特定パラメータの制御
====

Sensitive Parameterの他、NAP WAFではより詳細な制御をすることが可能です。

1. 事前動作確認
----

`Tips1. アカウントの登録 <https://f5j-nginx-plus-lab2-security.readthedocs.io/en/latest/class1/module03/module03.html#tips1>`__ の手順に従ってテスト用アカウントを作成してください。

作成後、画面右上 ``Account`` > ``test@example.com(作成したユーザのメールアドレス)`` をクリックし、 ``User Profile`` を開いてください。
ここではユーザの名称や、ユーザのアイコンとなる画像をアップロードすることが可能です。

ブラウザの右上 ``︙`` > ``More tools`` > ``Developer tools`` を開きます。
表示された ``Developer tools`` の ``Network`` タブを開き、通信の状況を取得します。

   .. image:: ./media/chrome-developer-tool.jpg
       :width: 400

| ``Username`` に ``dummyname`` を入力し、 ``Set Username`` をクリックし、動作を確認してください。
| ``Developer tools`` の検索窓に ``profile`` と入力し、 ``Status`` が ``30x`` の行をクリックしてください

   .. image:: ./media/chrome-setusername.jpg
       :width: 400

``Payload`` のタブを開くと、Form Data で ``username: dummyname`` が送付されていることが確認できます。
このUsername欄は、様々な特殊文字など自由に入力することが可能です。入力値を意図した様に制御するようWAFのポリシーを設定します。

2. 設定
----

| Parameterの設定を行います。
| WAFのセキュリティポリシーを変更し、設定を反映します

| Parameterのセキュリティポリシーは様々なVIOLATIONが関係します。
| 制御の内容に応じてVIOLATIONを有効にする場合がありますので、サンプルをよく確認して設定してください

- 設定サンプル: `User-Defined Parameters <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/#user-defined-parameters>`__
- VIOLATIONの意味: `Supported Violations <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/#supported-violations>`__

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l6_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-20 10:00:50.107946293 +0900
   +++ custom_policy.json  2022-04-21 00:27:37.705482111 +0900
   @@ -1,9 +1,93 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "parameter",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +       "blocking-settings": {
   +           "violations": [
   +               {
   +                   "name": "VIOL_PARAMETER_STATIC_VALUE",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_VALUE_LENGTH",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_NUMERIC_VALUE",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_DATA_TYPE",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_VALUE_METACHAR",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_NAME_METACHAR",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_VALUE_BASE64",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_MULTIPART_NULL_VALUE",
   +                   "alarm": true,
   +                   "block": true
   +               },
   +               {
   +                   "name": "VIOL_PARAMETER_LOCATION",
   +                   "alarm": true,
   +                   "block": true
   +               }
   +            ]
   +        },
   +        "parameters": [
   +            {
   +                "name": "username",
   +                "type": "explicit",
   +                "valueType": "user-input",
   +               "dataType": "alpha-numeric",
   +               "decodeValueAsBase64": "required",
   +                "parameterLocation": "form-data",
   +               "checkMaxValueLength": true,
   +               "checkMinValueLength": true,
   +               "maximumLength": 5,
   +               "minimumLength": 0,
   +                "urls": [
   +                    {
   +                        "method": "*",
   +                        "name": "/profile",
   +                        "type": "explicit",
   +                        "wildcardOrder": 1
   +                    }
   +               ]
   +            }
   +        ],
   +       "response-pages": [
   +            {
   +                "responseContent": "Attack is detected ID: <%TS.request.ID()%><br>Redirect in 5 sec... <script>setTimeout(function(){location.href='/profile'},5000);</script>",
   +                "responseHeader": "HTTP/1.1 200",
   +                "responseActionType": "custom",
   +                "responsePageType": "default"
   +            }
   +        ]
        }
    }

- ``blocking-settings`` の ``violations`` で、パラメータ制御に関連する VIOLATION を有効にしています
- ``parameters`` でパラメータ制御の設定をしています。今回対象となるパラメータは ``username`` で ``最大文字数(maximumLength)`` 、 入力値を ``アルファベットと数字(dataType)`` を指定しています。
- ``response-pages`` で エラーページを指定します。条件で拒否対象と判定された場合、自動的にリダイレクトするようにしています

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

3. 動作確認
----

1. 長い名前の入力

``Username`` に ``longname`` を入力します。
エラーページは5秒のみの表示となりますので、Support ID を取得する場合には注意ください。

   .. image:: ./media/chrome-setusername-longname.jpg
       :width: 400

ログを確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。

``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。

   .. image:: ./media/elk-discover-waflogs.jpg
       :width: 400

画面上部の検索窓に ``support_id **画面に表示されたsupport ID**`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-longname.jpg
       :width: 400

該当の通信が表示されています。

- ``username`` に ``longname`` が入力されています
- ``violations`` に、 ``Illegal parameter value length`` と表示されています

この様に、文字列の長さを指定することで想定外の入力値を制御することが可能です。


2. 許可されない文字の入力

``Username`` に 許可されない文字列 ``a@!b`` を入力します。
エラーページは5秒のみの表示となりますので、Support ID を取得する場合には注意ください。

   .. image:: ./media/chrome-setusername-nonalphanum.jpg
       :width: 400

ログを確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。

``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。
画面上部の検索窓に ``support_id **画面に表示されたsupport ID**`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-nonalphanum.jpg
       :width: 400

該当の通信が表示されています。
ログの内容を確認すると、 ``username`` に ``a@!b`` が入力され、 ``violations`` に、 ``Illegal Base64 value`` と表示されていることがわかります。

この様に、文字列のタイプを指定することで想定外の入力値を制御することが可能です。


7. Bot Clientの確認
====

NAP WAFはBot Signatureを持ち、クライアントが操作するツール以外に、Google等に代表される各サイトをクロールするツールに置いても判定・制御することが可能です。

1. 設定
----

Botの設定を行います。
WAFのセキュリティポリシーを変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l7_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-20 10:00:50.107946293 +0900
   +++ custom_policy.json  2022-04-21 00:49:07.276916732 +0900
   @@ -1,9 +1,30 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "bot-signature",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +       "bot-defense": {
   +            "settings": {
   +                "isEnabled": true
   +            },
   +            "mitigations": {
   +                "classes": [
   +                    {
   +                        "name": "trusted-bot",
   +                        "action": "alarm"
   +                    },
   +                    {
   +                        "name": "untrusted-bot",
   +                        "action": "block"
   +                    },
   +                    {
   +                        "name": "malicious-bot",
   +                        "action": "block"
   +                    }
   +                ]
   +            }
   +        }
        }
    }



このラボでCurlコマンドによる疎通確認を行っている箇所があります。

| デフォルトのセキュリティポリシーであればCurlコマンドはブロックされず、コンテンツの取得が可能です。
| ただし、Curlはブラウザなどとは異なり ``Untrusted Bot`` というクラスに分類されます。

今回の設定では、 ``Untrusted Bot`` を ``Block`` に変更します。

疎通確認で、 ``Curl`` と ``ブラウザ`` の双方から接続を行い、挙動を確認します

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

2. 動作確認
----

Curl コマンドを使ってリクエストを送信します。 

.. code-block:: cmdin

  curl -s "localhost/" | grep title

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  <html>
      <head>
        <title>Request Rejected</title>
      </head>
      <body>The requested URL was rejected. Please consult with your administrator.<br><br>
        Your support ID is: 16452723180063667004<br><br>
        <a href='javascript:history.back();'>[Go Back]</a>
      </body>
  </html>


ログを確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。

``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。
画面上部の検索窓に ``support_id **画面に表示されたsupport ID**`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-botsig.jpg
       :width: 400

該当の通信が表示されています。

- ``bot_category`` が ``HTTP Library`` 、 ``bot_signature_name`` が ``curl`` 、 ``client_class`` が ``Untrusted Bot`` となっています
- ``outcome`` が ``REJECTED`` となっています。このことから通信が拒否されたことがわかります
- ``violations`` が ``Bot Client Detected`` となっていることから、Botの判定によって通信が拒否されたと判断できます


次にブラウザでTopページにアクセスしてください


ブラウザの場合、通信はエラーなく終了しました。

ログを確認します。
画面上部の検索窓に ``client_class Browser`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-botsig.jpg
       :width: 400

ブラウザを通じてアクセスした場合、様々な通信が行われます。
今回はBotとの比較が主な目的となりますので、適当なログを選択いただければ問題ありません。

- ``bot_***`` に関する項目に情報がなく、 ``N/A`` となっています
- ``client_application`` が ``Chrome`` 、 ``client_class`` が ``Browser`` となっています

この様に、Curlコマンドとブラウザで接続した場合にはそれぞれ別のクライアントであることが識別されており、
Bot Signatureを利用することでNAP WAFが持つBot Signatureの機能により通信の制御が可能です


8. IPアドレスによる制御
====

IPアドレスによる制御を行い、NAP WAFの機能で制御を行い、同様にログを確認することができます。

1. 設定
----

Botの設定を行います。
WAFのセキュリティポリシーを変更し、設定を反映します

.. code-block:: cmdin

   # sudo su
   # cd /etc/nginx/conf.d
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_demo.conf default.conf
   # cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l1_custom_log_format.json custom_log_format.json
   cp ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l8_custom_policy.json custom_policy.json

WAFを設定を確認します

今回確認するポリシーについて前回の内容との差分を確認します。

.. code-block:: cmdin

   diff -u ~/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json custom_policy.json

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

   --- /root/f5j-nginx-plus-lab2-security-conf/waf/waf-l2_custom_policy.json     2022-04-20 10:00:50.107946293 +0900
   +++ custom_policy.json  2022-04-21 08:38:00.828843367 +0900
   @@ -1,9 +1,16 @@
    {
        "policy":
        {
   -        "name": "blocking",
   +        "name": "iplist",
            "template": { "name": "POLICY_TEMPLATE_NGINX_BASE" },
            "applicationLanguage": "utf-8",
   -        "enforcementMode": "blocking"
   +        "enforcementMode": "blocking",
   +        "whitelist-ips": [
   +            {
   +                "blockRequests": "always",
   +                "ipAddress": "10.0.0.0",
   +                "ipMask": "255.0.0.0"
   +            }
   +        ]
        }
    }

| ``whitelist-ips`` でIPアドレスを指定した通信の制御を行います。
| ``blockRequests`` で以下パラメータを指定することにより、指定したIPアドレスに対し通信を制御することが可能です。

+---------------+--------------+-------------------------------------------------------------------------------+
|               |指定する値    |動作                                                                           |
+---------------+--------------+-------------------------------------------------------------------------------+
|Policy Default |policy-default|指定したIPからの通信をBlocking Settingの内容に従って処理する                   |
+---------------+--------------+-------------------------------------------------------------------------------+
|Never Block    |never         |指定したIPからの通信を許可する。Policyのその他設定よりこちらの処理が優先される |
+---------------+--------------+-------------------------------------------------------------------------------+
|Always Block   |always        |指定したIPからの通信をブロックする                                             |
+---------------+--------------+-------------------------------------------------------------------------------+

プロセスを再起動し、設定を反映します

.. code-block:: cmdin

  nginx -s reload

2. 動作確認
----

Curl コマンドを使ってリクエストを送信します。 

.. code-block:: cmdin

  curl -s "localhost/" | grep title

.. code-block:: bash
  :caption: 実行結果サンプル
  :linenos:

  <title>OWASP Juice Shop</title>

Curlコマンドではローカルホストへアクセスしており、正常が完了したことを確認できます。

ブラウザでアクセスします。通信がブロックされました。

   .. image:: ./media/chrome-ips-blocked.jpg
       :width: 400


ログを確認します。 ``Jump Host`` でブラウザを起動し、 ``http://elk:5601`` を開いてください。

``ELK`` > ``Discover`` > ``waf-logs-*`` を開きます。

   .. image:: ./media/elk-discover-waflogs.jpg
       :width: 400

画面上部の検索窓に ``support_id **画面に表示されたsupport ID**`` と入力し ``Enter`` を押してください。

   .. image:: ./media/elk-discover-ips.jpg
       :width: 400

該当の通信が表示されています。
- ``ip_client`` が ``10.1.1.4`` となっており設定の内容に該当することがわかります
- ``outcome`` が ``REJECTED`` となり通信が拒否されています
- ``violations`` に IPアドレスがリストに含まれていると記載があります

特定のIPアドレスに対して制御することができます

9. Signatureのアップデート
====

NAP WAFのSignatureは頻繁にアップデートされます。
Signatureのアップデートはインストール時に確認いただいた通り、各種OSのパッケージ管理コマンドを通じて管理を頂きます。

以下が参考の確認結果です。

.. code-block:: bash
  :linenos:

   # dpkg-query -l | grep app-protect

   ii  app-protect                        25+3.671.0-1~focal                    amd64        App-Protect package for Nginx Plus, Includes all of the default files and examples. Nginx App Protect provides web application firewall (WAF) security protection for your web applications, including OWASP Top 10 attacks.
   ii  app-protect-attack-signatures      2021.11.16-1~focal                    amd64        Attack Signature Updates for App-Protect
   ii  app-protect-common                 8.12.1-1~focal                        amd64        NGINX App Protect
   ii  app-protect-compiler               8.12.1-1~focal                        amd64        Control-plane(aka CP) for waf-general debian
   ii  app-protect-dos                    25+2.0.1-1~focal                      amd64        Nginx DoS protection
   ii  app-protect-engine                 8.12.1-1~focal                        amd64        NGINX App Protect
   ii  app-protect-plugin                 3.671.0-1~focal                       amd64        NGINX App Protect plugin

このコマンドの出力結果では、Signatureは ``2021/11/16`` のものであることがわかります

インストール可能なパッケージを Ubuntu で確認する方法は以下です。

.. code-block:: bash
  :linenos:

   # apt list app-protect -a
   Listing... Done
   app-protect/stable,now 26+3.796.0-1~focal amd64 [installed]
   app-protect/stable 26+3.780.1-1~focal amd64
   app-protect/stable 25+3.760.0-1~focal amd64
   app-protect/stable 25+3.733.0-1~focal amd64
   app-protect/stable 25+3.671.0-1~focal amd64
   app-protect/stable 24+3.639.0-1~focal amd64
   app-protect/stable 24+3.612.0-1~focal amd64
   app-protect/stable 24+3.583.0-1~focal amd64
   app-protect/stable 24+3.512.0-1~focal amd64
   app-protect/stable 23+3.462.0-1~focal amd64
   
   # apt list app-protect-attack-signatures -a
   Listing... Done
   app-protect-attack-signatures/stable,now 2022.04.10-1~focal amd64 [installed]
   app-protect-attack-signatures/stable 2022.03.31-1~focal amd64
   app-protect-attack-signatures/stable 2022.03.23-1~focal amd64
   app-protect-attack-signatures/stable 2022.03.15-1~focal amd64
   app-protect-attack-signatures/stable 2022.03.02-1~focal amd64
   app-protect-attack-signatures/stable 2022.02.24-1~focal amd64
   app-protect-attack-signatures/stable 2022.02.15-1~focal amd64
   app-protect-attack-signatures/stable 2022.02.08-1~focal amd64
   app-protect-attack-signatures/stable 2022.02.01-1~focal amd64
   app-protect-attack-signatures/stable 2022.01.25-1~focal amd64
   app-protect-attack-signatures/stable 2022.01.12-1~focal amd64
   app-protect-attack-signatures/stable 2022.01.04-1~focal amd64
   ** 省略 **

| NAP WAFのSignatureは、そのSignatureがリリースした時点にサポートされるNGINX PlusとNAP WAF、そしてその後にリリースされるNGINX PlusとNAP WAFで動作します。
| つまり、Signatureがリリースした時点の前にサポートされていたNGINX PlusとNAP WAFでは動作しません。
| 最新のSignatureを利用するためには、その時点でサポートされているNGINX PlusとNAP WAFにアップグレードする必要があります。

参考に、Attack Signatureをダウングレードする手順を以下に示します

`NAP WAFのドキュメント <https://docs.nginx.com/nginx-app-protect/>`__ の ``Releases`` を見ると、このパッケージは ``March 9, 2022`` リリースされたものであることがわかります。

Versionを指定してSignatureをインストールします。(この例では、古いSignatureを選択します)

.. code-block:: bash
  :linenos:

  # apt install app-protect-attack-signatures=2022.03.15-1~focal

  ** 省略 **
  Unpacking app-protect-attack-signatures (2022.03.15-1~focal) over (2022.04.10-1~focal) ...
  Setting up app-protect-attack-signatures (2022.03.15-1~focal) ...
  In order for the signature update to take effect, NGINX must be reloaded.

  #  apt list app-protect-attack-signatures -a
  Listing... Done
  app-protect-attack-signatures/stable 2022.04.10-1~focal amd64 [upgradable from: 2022.03.15-1~focal]
  app-protect-attack-signatures/stable 2022.03.31-1~focal amd64
  app-protect-attack-signatures/stable 2022.03.23-1~focal amd64
  app-protect-attack-signatures/stable,now 2022.03.15-1~focal amd64 [installed,upgradable to: 2022.04.10-1~focal]
  app-protect-attack-signatures/stable 2022.03.02-1~focal amd64
  app-protect-attack-signatures/stable 2022.02.24-1~focal amd64
  app-protect-attack-signatures/stable 2022.02.15-1~focal amd64
  ** 省略 **

NGINXを再起動し、Signatureが反映されたことを確認します

.. code-block:: bash
  :linenos:
 
   # nginx -s reload
   # grep attack_signatures_package /var/log/nginx/error.log  | tail -2
   2022/04/22 08:18:20 [notice] 8423#8423: APP_PROTECT { "event": "configuration_load_success", "software_version": "3.796.0", "completed_successfully":true,"attack_signatures_package":{"version":"2022.04.10","revision_datetime":"2022-04-10T12:51:45Z"},"threat_campaigns_package":{},"software_version":""}
   2022/04/22 09:27:35 [notice] 8452#8452: APP_PROTECT { "event": "configuration_load_success", "software_version": "3.796.0", "completed_successfully":true,"attack_signatures_package":{"version":"2022.03.15","revision_datetime":"2022-03-15T11:35:54Z"},"threat_campaigns_package":{},"software_version":""}

2行目が該当のログとなります。1行目と比較すること ``attack_signatures_package`` の ``version`` の項目に変更後のVersionが記載されたことがわかります。

10. NAP WAF のアップグレード
====

ここでは、NAP WAFのアップグレードについてまとめます。

| NAP WAFはアップグレードすることが可能です。
| NAP WAFを古いVersionから新しいVersionにアップグレードされる場合、NAP WAFは一度削除され、再度新しいVersionがインストールされます。
| 同時に古いデフォルトPolicyが削除され、新しいデフォルトPolicyが配置されます。独自にポリシーの設定を行っている場合には、NGINXの設定ファイルに従って動作します。

NGINXのバージョンのみをアップグレードした場合、NAP WAFはアンインストールされるため、再度 NAP WAFのインストールが必要となります。
NAP WAFのインストール後、以下コマンドを参考にNGINXの再起動を行ってください。

.. code-block:: bash
  :linenos:

  # sudo su
  systemctl restart nginx

11. Threat Campaign Signature
====

NAP WAFは Threat Campaign という機能を持っています。
これは、F5のセキュリティチームで観測した、攻撃のキャンペーン(主要な攻撃の手口)の一連のふるまいから同種の攻撃であることを特定し高い精度で検知、制御する機能です。
Attack Signatureは各通信のデータを対象とするのに対し、この機能は攻撃開始から複数の通信に渡るまでそのふるまいで判定する点が違いとなります。

詳細は `Threat Campaigns <https://docs.nginx.com/nginx-app-protect/configuration-guide/configuration/#threat-campaigns>`__ をご覧ください。

Threat Campaign の導入方法を紹介します。

NGINX App Protect WAF を導入した際に、必要となる場合は Threat Campaign Signature を個別にインストールする必要があります。

.. code-block:: bash
  :linenos:

   # sudo su
   apt-get install app-protect-threat-campaigns

インストールが完了した後、パッケージを確認します

.. code-block:: bash
  :linenos:

   # dpkg-query -l | grep app-protect
   ii  app-protect                        26+3.796.0-1~focal                    amd64        App-Protect package for Nginx Plus, Includes all of the default files and examples. Nginx App Protect provides web application firewall (WAF) security protection for your web applications, including OWASP Top 10 attacks.
   ii  app-protect-attack-signatures      2022.04.10-1~focal                    amd64        Attack Signature Updates for App-Protect
   ii  app-protect-common                 10.29.1-1~focal                       amd64        NGINX App Protect
   ii  app-protect-compiler               10.29.1-1~focal                       amd64        Control-plane(aka CP) for waf-general debian
   ii  app-protect-dos                    26+2.2.20-1~focal                     amd64        Nginx DoS protection
   ii  app-protect-engine                 10.29.1-1~focal                       amd64        NGINX App Protect
   ii  app-protect-plugin                 3.796.0-1~focal                       amd64        NGINX App Protect plugin
   ii  app-protect-threat-campaigns       2022.03.30-1~focal                    amd64        Threat Campaign Updates for App-Protect

プロセスを再起動します。

.. code-block:: bash
  :linenos:

   # nginx -s reload

Threat Campaign Signatureを確認します。読み込むまでに数分程度時間がかかる場合があります

.. code-block:: bash
  :linenos:

   # grep attack_signatures_package /var/log/nginx/error.log  | tail -2
   2022/04/22 11:12:43 [notice] 8452#8452: APP_PROTECT { "event": "configuration_load_success", "software_version": "3.796.0", "completed_successfully":true,"attack_signatures_package":{"version":"2022.04.10","revision_datetime":"2022-04-10T12:51:45Z"},"threat_campaigns_package":{},"software_version":""}
   2022/04/22 11:16:33 [notice] 8452#8452: APP_PROTECT { "event": "configuration_load_success", "software_version": "3.796.0", "completed_successfully":true,"attack_signatures_package":{"version":"2022.04.10","revision_datetime":"2022-04-10T12:51:45Z"},"threat_campaigns_package":{"version":"2022.04.20","revision_datetime":"2022-04-20T14:14:04Z"},"software_version":""}

2行目が該当のログとなります。1行目と比較すること ``threat_campaigns_package`` の項目にVersionが記載されたことがわかります。
このように設定することで Threat Campaigns Signature を利用することが可能です。

Tips1. アカウントの登録
====

OWASP Juice Shopではアカウントの登録が可能です。ログインいただくと、カートへ商品の追加などの操作をいただくことが可能です。

1. アカウントの登録
----

動作確認の為、 ``OWASP Juice SHop`` に新規アカウントを登録します。

``Jump Host`` より ``Owasp Juice Shop`` にアクセスします。ブラウザを起動し、 ``http://juice-shop`` を開いてください

画面右上 ``Account`` > ``Login`` をクリックします。
すでに別のアカウントでログインしている場合、一度ログアウトをしてください。

画面中段 「Not yet a customer?」をクリックし、以下の内容を入力してください。

   .. image:: ./media/owasp-js-new-account.jpg
       :width: 400

+-------------------+------------------------+
|Email              | ``test@example.com``   |
+-------------------+------------------------+
|Password           | ``testuser``           |
+-------------------+------------------------+
|Repeat Password    | ``testuser``           |
+-------------------+------------------------+
|Security Questions | ``自由に指定下さい``   |
+-------------------+------------------------+
|Answer             | ``自由に指定下さい``   |
+-------------------+------------------------+

(説明の用途でメールアドレス・パスワードを指定していますがいずれの文字列でも問題ありません)


再度表示されますログイン画面に作成したアカウントの情報を入力し、ログインしてください。

   .. image:: ./media/owasp-js-new-account2.jpg
       :width: 400

アカウント情報を確認すると指定のメールアドレスでログインしたことが確認できます。

   .. image:: ./media/owasp-js-new-account3.jpg
       :width: 400







