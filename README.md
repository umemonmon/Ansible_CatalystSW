Ansible CatalystSW プロジェクト

このリポジトリは、Cisco Catalyst 9000 シリーズスイッチの構築・運用を効率化するための Ansible プレイブック群をまとめたプロジェクトです。各プレイブックは特定の運用タスクを自動化するよう設計されており、対象機器の OS バージョンやライセンス方式に応じて使い分けます。

ディレクトリ構成（例）

以下は \wsl.localhost\Ubuntu-26.04\home\XXXX\Ansible\01_Catalyst フォルダの一般的な構成例です。環境により多少異なる場合があります。

パス	説明
FiremwareUpdate_Playbook.yml	Catalyst スイッチのファームウェアを更新するプレイブック。機種に適したイメージをリモートサーバから取得し、適用後に再起動までを自動化します。

LicenseOfflineInit_Playbook.yml	Smart Licensing Using Policy の CSSM オフライン方式に対応したライセンス設定プレイブック。ライセンスブートレベルの確認・変更、Smart Transport の無効化、RUM レポート生成、ACK ファイルのインポートなどを自動化します。

group_vars/	グループ単位の変数ファイルを配置するディレクトリ。例えば C9000.yml には Catalyst 9000 用の接続情報やライセンス変数を定義します。

host_vars/	個別ホスト向け変数を格納するディレクトリ。ホスト名と同名の YAML ファイルを置くことで、そのホスト固有の IP アドレスやパラメータを指定できます。

inventory	対象機器を定義するインベントリファイル。[CatalystSW] グループにスイッチの管理用 IP を列挙します。

roles/	再利用可能なロールを格納するディレクトリ。共通のタスクやテンプレートをロールに切り出すことでプレイブックを簡潔に保ちます。

ansible.cfg	Ansible の動作設定ファイル。接続方式やコレクションパスなどをここで定義します。
主なプレイブック
FiremwareUpdate_Playbook.yml
Catalyst 9000 シリーズの IOS-XE イメージを更新する際に使用するプレイブックです。主な機能は次のとおりです。
接続チェック: インベントリ内の各デバイスに対して SSH 接続の可否を確認し、接続可能なホストのみ更新処理を実施します。
イメージ転送: 指定された TFTP/HTTP サーバから新しいファームウェアイメージを取得し、スイッチのフラッシュへコピーします。
ブート設定と再起動: 新イメージをブート変数に設定し、必要に応じて再起動してアップグレードを完了します。
確認: 再起動後に show version などでバージョンを確認し、アップグレードが成功したかをレポートします。
LicenseOfflineInit_Playbook.yml

Smart Licensing Using Policy の CSSM オフライン方式 でライセンス設定とレポーティングを自動化するためのプレイブックです。ブログ記事を参照しながら作成されています。

プレイブックの流れは次のとおりです。

SSH接続プリチェック: wait_for_connection モジュール等を用いて、SSH 接続できるホストのみを対象に処理を進めます。接続できないホストはスキップします。
ホスト名取得: ios_command モジュールで show running-config | include ^hostname を実行し、RUM/ACK ファイル名に使用するホスト名を取得します。
ライセンスレベル確認と変更: show running-config all | include license boot level、show license summary で現状を確認し、必要に応じて license boot level を設定します。
Smart Transport オフ: CSSM との通信を行わないため license smart transport off を設定します。
RUM レポート生成: license smart save usage all file flash:<hostname>_rum.txt コマンドでライセンス使用状況レポートを生成し、Ansible 制御ノードへコピーします。
ACK ファイル準備とインポート: RUM ファイルを CSSM ポータルにアップロードして ACK ファイルをダウンロードする操作は手動です。ダウンロード後、ACK ファイル（ACK_<hostname>_rum.txt）を指定の TFTP サーバに配置し、プレイブックがスイッチに取得・インポートします。
結果確認: show license summary や show license status でライセンスステータスが IN USE になっており、Last ACK received が更新されていることを確認します。
実行方法の例
# インベントリと変数の準備
$ cat inventory
[CatalystSW]
10.0.0.1
10.0.0.2

$ cat group_vars/CatalystSW.yml
ansible_network_os: cisco.ios.ios
ansible_user: myuser
ansible_password: mypassword

# プレイブック実行
$ ansible-playbook -i inventory FiremwareUpdate_Playbook.yml
$ ansible-playbook -i inventory LicenseOfflineInit_Playbook.yml

注意点
ファームウェア更新やライセンス設定はネットワークに影響を与える可能性があるため、メンテナンス時間帯に実施してください。
ライセンスブートレベルを変更した場合は再起動が必要です。再起動前には冗長構成や経路切替を確認してください。
RUM ファイルのアップロードと ACK ファイルの取得は CSSM Web ポータルで手動で行います。ポータルの手順はブログ記事を参照してください。
本リポジトリの内容やコマンドは環境・IOS バージョンにより異なる場合があります。製品マニュアルやメーカ資料も必ず参照してください。
