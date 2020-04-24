[OCI] Oracle Functionsから OCI Vault シークレットを使った Autonomous Databaseに接続
https://blogs.oracle.com/developers/oracle-functions-connecting-to-an-atp-database-with-a-wallet-stored-as-secrets

事前準備
- Autonomous Dataabseの作成
    - オプション）sample.sqlで表、データを作成
- 接続ウォレットのダウンロード
- Functions 実行環境の整備（Cloud Shellでも可）

ステップ.1 シークレット操作用のポリシーの割り当て
- 以下のサービスレベルポリシーを適用
    - allow service VaultSecret to use vaults in tenancy
    - allow service VaultSecret to use keys in tenancy
- 以下のパーミッションが適用されたグループ内のユーザーで作業
    - allow group [group] to manage vaults in tenancy
    - allow group [group] to manage keys in tenancy

ステップ.2 ウォレットのシークレット作成
- ウォレット内の各ファイルとDBユーザのパスワードをBase64でエンコード
	- base64 -i cwallet.sso >  cwallet,sso.base64
    - base64 -i ewallet.p12 >  ewallet.p12.base64
    - base64 -i keystore.jks >  keystore.jks.base64
    - base64 -i ojdbc.properties >  ojdbc.properties.base64
    - base64 -i sqlnet.ora >  sqlnet.ora.base64
    - base64 -i tnsnames.ora >  tnsnames.ora.base64
    - base64 -i truststore.jks >  truststore.jks.base64
    - base64 -i password.txt >  password.txt.base64

    ※ password.txt には DBユーザのパスワードを記入

- OCI Webコンソールから「セキュリティ」＞「ボールト」から
    - 「ボールト作成」、作成したボールトに「キーの作成」
- Base64でエンコードした各ファイルごとにシークレットを作成
    - 名前・説明を入力
    - 暗号化キー：作成したキー
    - シークレット・タイプ：Base64
    - シークレット・コンテンツ
        - Base64でエンコードしたファイルの中身をペースト

- 作成したシークレットごとにOCIDをコピー

ステップ.3 Functionの作成・動作確認
- アプリケーションの作成
    - fn create app oci-adb-jdbc-java-app --annotation oracle.com/oci/subnetIds='["ocid1.subnet.oc1.phx..."]'
- アプリケーションに構成を追加
    - シークレットのOCID
        - fn config app oci-adb-jdbc-java-app CWALLET_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app EWALLET_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app KEYSTORE_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app OJDBC_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app SQLNET_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app TNSNAMES_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app TRUSTSTORE_ID ocid1.vaultsecret.oc1.iad..

    ADBへの接続用のDBユーザ名・パスワードのシークレットOCID・接続文字列
        - fn config app oci-adb-jdbc-java-app DB_USER [user]
        - fn config app oci-adb-jdbc-java-app PASSWORD_ID ocid1.vaultsecret.oc1.iad...
        - fn config app oci-adb-jdbc-java-app DB_URL jdbc:oracle:thin:\@[tns name]\?TNS_ADMIN=/tmp/wallet
    - 実行するSQL文
        - fn config app ziptodd SQL_TEXT "select * from employees"
- ファンクションの作成/テストディレクトリの削除
    - fn init --runtime java oci-adb-jdbc-java-secrets
    - cd oci-adb-jdbc-java-secrets
    - rm -r src/test/

    ファンクションの作成
    - pom.xmlファイル
    - func.yamlファイル
    - HelloFunction.javaファイル
    の編集（本ファイルに置き換え）

        - HelloFunction.java の「secretsClient.setRegion(Region.US_ASHBURN_1);」行を利用リージョンに変更
        - 例）東京リージョンの場合
        -     secretsClient.setRegion(Region.AP_TOKYO_1);

- ファンクションのデプロイと呼び出し
    - fn deploy --app oci-adb-jdbc-java-app
    - fn invoke oci-adb-jdbc-java-app oci-adb-jdbc-java-secrets

