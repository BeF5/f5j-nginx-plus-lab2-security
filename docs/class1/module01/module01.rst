環境
#######

実施環境
========

-  事前にラボ環境へのInviteを行っておりますので、メールをご確認ください
-  利用するコマンド： git , jq , sudo, curl
-  NGINX Trialライセンスの取得、ラボ実施ユーザのHome Directoryへ配置

ラボ環境 (UDF(Unified Demonstration Framework)) コンポーネントへの接続
======================================================================

| 弊社が提供するラボ環境を使って動作を確認いただきます。
| ラボ環境を起動する等、一部ブラウザを使って操作します。
| Google ChromeがSupportブラウザとなります。その他ブラウザでは正しく動作しない場合があることご了承ください。
| 参照：\ `UDF Supported Browsers and Clients <https://help.udf.f5.com/en/articles/3470266-supported-browsers-and-clients>`__

ラボ環境構成図
----

   .. image:: ./media/udflab_basic_Lab_jp.png
      :width: 400

RDP接続
----

a. Windows Jump HostへのRDP接続
~~~~

.. NOTE::
   端末のセキュリティ設定等により、RDPクライアントによる接続が出来ない場合、 `b. Windows Jump HostへVNCで接続 <#b-windows-jump-hostvnc>`__ を参照してください


Windows Jump HostからCLIの操作を行う場合、以下タブからRDP Clientファイルをダウンロードいただき接続ください

   .. image:: ./media/udf_jumpbox.png
      :width: 200

.. NOTE::
   | RDPのUser名、パスワードはDETAILSをクリックし、GeneralのタブのCredentialsの項目を参照ください
   | ``user`` でログインしてください 

   - .. image:: ./media/udf_jumpbox_loginuser.png
       :width: 200
    
   - .. image:: ./media/udf_jumpbox_loginuser2.png
       :width: 200


b. Windows Jump HostへVNCで接続
~~~~

vnc-windowsの ``vnc-win`` をクリックしてください

   .. image:: ./media/udf_vnc_jumpbox.png
      :width: 200

``接続`` をクリックしてください

   .. image:: ./media/udf_vnc_jumpbox2.png
      :width: 400

パスワードが求められます。 ``admin`` と入力してください

   .. image:: ./media/udf_vnc_jumpbox3.png
      :width: 400

Windowsのログイン画面が表示されます。VNCのメニューより、 ``Ctrl+Alt+Delを送信`` をクリックします

   .. image:: ./media/udf_vnc_jumpbox4.png
      :width: 400

適切なユーザを選択し、パスワードを ``キーボードで入力`` してください。ログインの情報は `a. Windows Jump HostへのRDP接続 <#a-windows-jump-hostrdp>`__ のパスワード情報を確認してください

   .. image:: ./media/udf_vnc_jumpbox5.png
      :width: 400

初期状態では、画面の解像度が低い値の場合があります。以下手順を参考に環境にあわせて解像度を変更してください
デスクトップで右クリックから ``Display Settings`` を選択

   - .. image:: ./media/udf_vnc_display.png
      :width: 200

   - .. image:: ./media/udf_vnc_display2.png
      :width: 400

   - .. image:: ./media/udf_vnc_display3.png
      :width: 400

SSHの接続
----

Windows Jump Hostへログインいただくと、SSH
Clientのショートカットがありますので、そちらをダブルクリックし
``ubuntu01`` へ接続ください

   - .. image:: ./media/putty_icon.jpg
      :width: 50

   - .. image:: ./media/putty_menu.jpg
      :width: 200




