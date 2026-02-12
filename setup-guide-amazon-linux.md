# OpenStack ハンズオン環境構築ガイド（Amazon Linux 2023）

## 概要

Amazon Linux 2023上でDevStackを使用してOpenStack環境を構築し、OPCEL試験対策のハンズオンを実施するための完全ガイドです。

---

## 前提条件

### 推奨EC2インスタンススペック
- **インスタンスタイプ**: t3.xlarge以上（4vCPU, 16GB RAM推奨）
- **ストレージ**: 50GB以上
- **OS**: Amazon Linux 2023
- **セキュリティグループ**: SSH (22), HTTP (80), HTTPS (443), Horizon (80/443)

---

## 1. 環境準備

### システムアップデートとパッケージインストール

```bash
# システムのアップデート
sudo dnf update -y

# 必要なパッケージのインストール
sudo dnf install -y git python3 python3-pip python3-devel gcc make

# 開発ツールのインストール
sudo dnf groupinstall -y "Development Tools"

# その他の依存パッケージ
sudo dnf install -y libffi-devel openssl-devel libxml2-devel libxslt-devel \
    libjpeg-devel zlib-devel mariadb-devel postgresql-devel
```

### stackユーザーの作成

DevStackは非rootユーザーで実行する必要があります。

```bash
# stackユーザーの作成
sudo useradd -s /bin/bash -d /opt/stack -m stack

# sudoers権限の付与
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

# stackユーザーに切り替え
sudo su - stack
```

---

## 2. DevStackのインストール

### DevStackのクローン

```bash
# stackユーザーとしてログインしていることを確認
cd ~
git clone https://opendev.org/openstack/devstack
cd devstack
```

### local.confの作成

DevStackの設定ファイルを作成します。

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# 管理者パスワード（すべて同じに設定）
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# ホスト設定
HOST_IP=$(hostname -I | awk '{print $1}')

# ログ設定
LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True

# サービス設定
# 基本サービスのみ有効化（最小構成）
disable_all_services
enable_service key mysql rabbit

# Horizon（ダッシュボード）
enable_service horizon

# Nova（コンピュート）
enable_service n-api n-cpu n-cond n-sch n-novnc n-api-meta

# Glance（イメージ）
enable_service g-api g-reg

# Neutron（ネットワーク）
enable_service q-svc q-agt q-dhcp q-l3 q-meta neutron

# Neutron設定
Q_USE_SECGROUP=True
FLOATING_RANGE="192.168.27.0/24"
FIXED_RANGE="10.0.0.0/24"
Q_FLOATING_ALLOCATION_POOL=start=192.168.27.100,end=192.168.27.200
PUBLIC_NETWORK_GATEWAY="192.168.27.1"
Q_USE_PROVIDERNET_FOR_PUBLIC=True
OVS_PHYSICAL_BRIDGE=br-ex
PUBLIC_INTERFACE=eth0
OVS_BRIDGE_MAPPINGS=public:br-ex

# イメージの自動ダウンロード
DOWNLOAD_DEFAULT_IMAGES=True
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"

# その他の設定
RECLONE=yes
EOF
```

### 最小構成版のlocal.conf（トラブル時用）

メモリやストレージが限られている場合は、こちらを使用してください。

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# パスワード設定
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# ホスト設定
HOST_IP=$(hostname -I | awk '{print $1}')

# ログ設定
LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=False

# 最小限のサービス
disable_all_services
enable_service key mysql rabbit
enable_service n-api n-cpu n-cond n-sch
enable_service g-api
enable_service q-svc q-agt q-dhcp q-l3 q-meta

# イメージ設定
DOWNLOAD_DEFAULT_IMAGES=True
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"

RECLONE=yes
EOF
```

### DevStackのインストール実行

```bash
# インストール実行（30分〜1時間程度かかります）
./stack.sh
```

インストールが成功すると、以下のような出力が表示されます：

```
=========================
DevStack Component Timing
=========================
This is your host IP address: xxx.xxx.xxx.xxx
This is your host IPv6 address: ::1
Horizon is now available at http://xxx.xxx.xxx.xxx/dashboard
Keystone is serving at http://xxx.xxx.xxx.xxx/identity/
```

---

## 3. インストール後の確認

### 環境変数の設定

```bash
# admin権限での認証情報を読み込む
source ~/devstack/openrc admin admin

# 環境変数の確認
env | grep OS_
```

### OpenStackクライアントの動作確認

```bash
# サービス一覧の確認
openstack service list

# エンドポイント一覧の確認
openstack endpoint list

# コンピュートサービスの確認
openstack compute service list

# ネットワーク一覧の確認
openstack network list

# イメージ一覧の確認
openstack image list
```

---

## 4. トラブルシューティング

### インストールが失敗した場合

```bash
# スタックのクリーンアップ
cd ~/devstack
./unstack.sh
./clean.sh

# 再度インストール
./stack.sh
```

### ログの確認方法

```bash
# stack.shのログ
tail -f ~/devstack/logs/stack.sh.log

# 各サービスのログ
sudo journalctl -u devstack@* -f

# エラーログのみを表示
sudo journalctl -u devstack@* -p err
```

### サービスの状態確認

```bash
# すべてのDevStackサービスの状態
sudo systemctl status "devstack@*"

# 特定のサービスの状態
sudo systemctl status devstack@n-api
sudo systemctl status devstack@q-svc
```

### サービスの再起動

```bash
# すべてのサービスを再起動
sudo systemctl restart devstack@*

# 特定のサービスのみ再起動
sudo systemctl restart devstack@n-api
```

---

## 5. 再起動後の起動手順

EC2インスタンスを再起動した場合、以下の手順でOpenStackを再起動します。

```bash
# stackユーザーに切り替え
sudo su - stack

# DevStackディレクトリに移動
cd ~/devstack

# サービスの再開（rejoin-stack.shを使用）
./rejoin-stack.sh
```

または、手動でサービスを起動する場合：

```bash
# すべてのDevStackサービスを起動
sudo systemctl start devstack@*

# 認証情報の読み込み
source ~/devstack/openrc admin admin

# サービスの確認
openstack service list
```

---

## 6. Horizonダッシュボードへのアクセス

### ブラウザからのアクセス

```
URL: http://<EC2のパブリックIP>/dashboard
ユーザー名: admin
パスワード: secret（local.confで設定した値）
ドメイン: default
```

### セキュリティグループの設定

EC2のセキュリティグループで以下のポートを開放してください：

- HTTP: 80
- HTTPS: 443（必要に応じて）
- SSH: 22（管理用）

---

## 7. 基本的なOpenStackコマンド

### 認証情報の読み込み

```bash
# 管理者として認証
source ~/devstack/openrc admin admin

# 一般ユーザーとして認証（demoプロジェクト）
source ~/devstack/openrc demo demo
```

### サービスの確認

```bash
# サービス一覧
openstack service list

# エンドポイント一覧
openstack endpoint list

# コンピュートホストの確認
openstack hypervisor list

# ネットワークエージェントの確認
openstack network agent list
```

---

## 8. 次のステップ

環境構築が完了したら、`openstack-commands.md`を参照して、以下のハンズオンを実施してください：

1. **Keystoneハンズオン**: ユーザー、プロジェクト、ロールの管理
2. **Glanceハンズオン**: イメージの管理
3. **Neutronハンズオン**: ネットワーク、サブネット、ルーターの作成
4. **Novaハンズオン**: インスタンスの起動と管理
5. **総合演習**: 完全な環境構築とインスタンスへのアクセス

---

## 9. 便利なエイリアス設定

```bash
# .bashrcに追加
cat >> ~/.bashrc << 'EOF'

# OpenStackエイリアス
alias os='openstack'
alias os-admin='source ~/devstack/openrc admin admin'
alias os-demo='source ~/devstack/openrc demo demo'
alias os-service='sudo systemctl status devstack@*'
alias os-restart='sudo systemctl restart devstack@*'
alias os-log='tail -f ~/devstack/logs/stack.sh.log'

EOF

# 設定を反映
source ~/.bashrc
```

使用例：
```bash
os-admin                    # 管理者として認証
os server list              # インスタンス一覧
os network list             # ネットワーク一覧
os-service                  # サービス状態確認
```

---

## 10. リソースのクリーンアップ

### DevStackの完全削除

```bash
cd ~/devstack
./unstack.sh
./clean.sh

# stackユーザーのホームディレクトリを削除（必要に応じて）
sudo rm -rf /opt/stack
```

---

## 付録A: Amazon Linux 2023固有の注意事項

### Python環境

Amazon Linux 2023ではPython 3.9以上が標準です。DevStackは自動的に適切なPythonバージョンを使用します。

### SELinux

デフォルトでSELinuxは有効ですが、DevStackはこれを考慮して動作します。問題が発生する場合：

```bash
# SELinuxの状態確認
getenforce

# 一時的に無効化（トラブルシューティング用）
sudo setenforce 0
```

### ファイアウォール

```bash
# firewalldの状態確認
sudo systemctl status firewalld

# DevStackはiptablesを使用するため、firewalldと競合する可能性があります
# 必要に応じて停止
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

---

## 付録B: パフォーマンス最適化

### スワップの設定

メモリが不足する場合、スワップを追加します：

```bash
# 4GBのスワップファイルを作成
sudo dd if=/dev/zero of=/swapfile bs=1G count=4
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 永続化
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### メモリ使用量の確認

```bash
# メモリ使用状況
free -h

# プロセスごとのメモリ使用量
ps aux --sort=-%mem | head -n 10
```

---

## OPCEL試験対策チェックリスト

- [ ] DevStackのインストール完了
- [ ] すべてのサービスが正常に起動
- [ ] Horizonダッシュボードにアクセス可能
- [ ] openstack CLIコマンドが実行可能
- [ ] ネットワークの作成と確認
- [ ] インスタンスの起動と停止
- [ ] フローティングIPの割り当て
- [ ] セキュリティグループの設定
- [ ] イメージの管理
- [ ] ユーザーとプロジェクトの管理

すべてのチェックが完了したら、`openstack-commands.md`の演習を実施してください！
