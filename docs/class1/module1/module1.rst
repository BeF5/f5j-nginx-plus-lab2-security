環境
#######

実施環境
========

-  事前にラボ環境へのInviteを行っておりますので、メールをご確認ください
-  利用するコマンド： git , jq , sudo, curl
-  NGINX Trialライセンスの取得、ラボ実施ユーザのHome Directoryへ配置

ラボ環境 (UDF(Unified Demonstration Framework)) コンポーネントへの接続
======================================================================

| 弊社が提供するLAB環境を使って動作を確認いただきます。
| ラボ環境を起動する等、一部ブラウザを使って操作します。
| Google
  ChromeがSupportブラウザとなります。その他ブラウザでは正しく動作しない場合があることご了承ください。
| 参照：\ `UDF Supported Browsers and
  Clients <https://help.udf.f5.com/en/articles/3470266-supported-browsers-and-clients>`__


Windows Jump HostへのRDP接続
----------------------------


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
   
Windows Jump Hostへログインいただくと、SSH
Clientのショートカットがありますので、そちらをダブルクリックし
``ubuntu01`` へ接続ください

   - .. image:: ./media/putty_icon.jpg
      :width: 50

   - .. image:: ./media/putty_menu.jpg
      :width: 200


Linux Hostへの接続 (Jump Host を利用しない場合)
-----------------------------------------------

| ``ubuntu01`` へのSSH接続は、Jump Host経由または、SSH鍵認証を用いて接続可能です。SSH鍵の登録手順は以下を参照ください
| **SSH鍵を登録頂いていない場合、SSHはグレーアウトします** 
| 手順： `UDF LAB SSH鍵登録マニュアル <https://github.com/hiropo20/partner_nap_workshop_secure/blob/main/UDF_SSH_Key.pdf>`_
| (こちらの手順が必要となる場合、本マニュアルを閲覧可に変更します)

