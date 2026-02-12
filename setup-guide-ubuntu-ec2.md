# OpenStack 環境構築ガイド（Ubuntu 22.04 on AWS EC2）

## システム情報
```
OS: Ubuntu 22.04 (Kernel 6.8.0)
環境: AWS EC2インスタンス
アーキテクチャ: x86_64
```

---

## 前提条件

### 推奨EC2インスタンススペック
- **インスタンスタイプ**: t3.xlarge 以上（4vCPU, 16GB RAM推奨）
- **ストレージ**: 60GB以上（80GB推奨）
- **OS**: Ubuntu 22.04 LTS
- **セキュリティグループ**: SSH (22), HTTP (80), HTTPS (443), OpenStack各種ポート

---

## ステップ1: システムの準備

### システムアップデート

```bash
# システムを最新状態に更新
sudo apt update
sudo apt upgrade -y

# システム再起動（カーネルアップデート時のみ）
sudo reboot
```

### 必要なパッケージのインストール

```bash
# 基本パッケージ
sudo apt install -y \
    git \
    python3 \
    python3-pip \
    python3-dev \
    build-essential \
    libssl-dev \
    libffi-dev \
    python3-venv

# その他の依存パッケージ
sudo apt install -y \
    curl \
    wget \
    vim \
    net-tools \
    bridge-utils
```

---

## ステップ2: stackユーザーの作成

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

## ステップ3: DevStackのインストール

### DevStackのクローン

```bash
# stackユーザーとしてログインしていることを確認
whoami  # 出力: stack

# ホームディレクトリに移動
cd ~

# DevStackをクローン
git clone https://opendev.org/openstack/devstack
cd devstack
```

### local.confの作成

OpenStack環境の設定ファイルを作成します。

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# パスワード設定（すべて同じに設定）
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# ホストIPの自動取得（EC2のプライベートIP）
HOST_IP=$(hostname -I | awk '{print $1}')

# ログ設定
LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True

# 基本サービスの有効化
disable_all_services

# Keystone（認証サービス）
enable_service key

# データベース
enable_service mysql

# メッセージキュー
enable_service rabbit

# Horizon（Webダッシュボード）
enable_service horizon

# Nova（コンピュートサービス）
enable_service n-api n-cpu n-cond n-sch n-novnc n-api-meta

# Glance（イメージサービス）
enable_service g-api

# Neutron（ネットワークサービス）
enable_service q-svc q-agt q-dhcp q-l3 q-meta neutron

# Neutronネットワーク設定
Q_USE_SECGROUP=True
FLOATING_RANGE="192.168.100.0/24"
FIXED_RANGE="10.0.0.0/24"
Q_FLOATING_ALLOCATION_POOL=start=192.168.100.100,end=192.168.100.200
PUBLIC_NETWORK_GATEWAY="192.168.100.1"

# イメージの自動ダウンロード
DOWNLOAD_DEFAULT_IMAGES=True
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"

# 再クローン設定
RECLONE=yes
EOF
```

### インストールの実行

```bash
# インストール開始（30分〜1時間程度かかります）
./stack.sh
```

インストール中は以下のようなログが表示されます：
```
This is your host IP address: 10.0.129.147
Horizon is now available at http://10.0.129.147/dashboard
Keystone is serving at http://10.0.129.147/identity/
```

---

## ステップ4: インストール後の確認

### 環境変数の設定

```bash
# 管理者権限での認証情報を読み込む
source ~/devstack/openrc admin admin

# 環境変数の確認
env | grep OS_
```

出力例：
```
OS_PROJECT_DOMAIN_ID=default
OS_REGION_NAME=RegionOne
OS_USER_DOMAIN_ID=default
OS_PROJECT_NAME=admin
OS_IDENTITY_API_VERSION=3
OS_PASSWORD=secret
OS_AUTH_TYPE=password
OS_AUTH_URL=http://10.0.129.147/identity
OS_USERNAME=admin
OS_TENANT_NAME=admin
```

### OpenStackサービスの確認

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

# ハイパーバイザーの確認
openstack hypervisor list
```

---

## ステップ5: Horizonダッシュボードへのアクセス

### セキュリティグループの設定

EC2のセキュリティグループで以下のポートを開放してください：

```
- SSH: 22 (管理用)
- HTTP: 80 (Horizon)
- HTTPS: 443 (Horizon - 必要に応じて)
- Keystone: 5000
- Nova: 8774
- Glance: 9292
- Neutron: 9696
```

### ブラウザからのアクセス

```
URL: http://<EC2のパブリックIP>/dashboard
ユーザー名: admin
パスワード: secret
ドメイン: default
```

---

## ステップ6: 基本的な動作確認

### イメージの確認

```bash
# イメージ一覧
openstack image list
```

出力例：
```
+--------------------------------------+---------+--------+
| ID                                   | Name    | Status |
+--------------------------------------+---------+--------+
| xxx-xxx-xxx-xxx                      | cirros  | active |
+--------------------------------------+---------+--------+
```

### ネットワークの確認

```bash
# ネットワーク一覧
openstack network list

# サブネット一覧
openstack subnet list
```

### フレーバーの確認

```bash
# フレーバー一覧
openstack flavor list
```

---

## ステップ7: 簡単な動作テスト

### インスタンスの起動テスト

```bash
# プライベートネットワークの確認
openstack network list

# インスタンスの起動
openstack server create \
  --flavor m1.tiny \
  --image cirros \
  --network private \
  test-instance

# インスタンスの状態確認
openstack server list

# インスタンスの詳細確認
openstack server show test-instance

# インスタンスの削除
openstack server delete test-instance
```

---

## ステップ8: 再起動後の起動手順

EC2インスタンスを再起動した場合、以下の手順でOpenStackを再起動します。

```bash
# stackユーザーに切り替え
sudo su - stack

# DevStackディレクトリに移動
cd ~/devstack

# サービスの再開
./rejoin-stack.sh

# または、手動でサービスを起動
sudo systemctl start devstack@*

# 認証情報の読み込み
source ~/devstack/openrc admin admin

# サービスの確認
openstack service list
```

---

## トラブルシューティング

### インストールが失敗した場合

```bash
# スタックのクリーンアップ
cd ~/devstack
./unstack.sh
./clean.sh

# 再度インストール
./stack.sh
```

### ログの確認

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
sudo systemctl status devstack@g-api
```

### サービスの再起動

```bash
# すべてのサービスを再起動
sudo systemctl restart devstack@*

# 特定のサービスのみ再起動
sudo systemctl restart devstack@n-api
sudo systemctl restart devstack@q-svc
```

### ディスク容量不足の場合

```bash
# ディスク使用状況の確認
df -h

# 不要なログの削除
sudo rm -rf /opt/stack/logs/*
sudo journalctl --vacuum-time=1d

# aptキャッシュのクリーンアップ
sudo apt clean
sudo apt autoremove -y
```

### メモリ不足の場合

```bash
# メモリ使用状況の確認
free -h

# スワップの作成（4GB）
sudo dd if=/dev/zero of=/swapfile bs=1G count=4
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 永続化
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 便利なエイリアス設定

```bash
# .bashrcに追加（ubuntuユーザーとして）
cat >> ~/.bashrc << 'EOF'

# OpenStackエイリアス
alias os='openstack'
alias os-stack='sudo su - stack'
alias os-admin='source /opt/stack/devstack/openrc admin admin'
alias os-demo='source /opt/stack/devstack/openrc demo demo'
alias os-service='sudo systemctl status devstack@*'
alias os-restart='sudo systemctl restart devstack@*'
alias os-log='tail -f /opt/stack/devstack/logs/stack.sh.log'
alias os-start='cd /opt/stack/devstack && ./rejoin-stack.sh'

EOF

# 設定を反映
source ~/.bashrc
```

使用例：
```bash
os-stack                    # stackユーザーに切り替え
os-admin                    # 管理者として認証
os server list              # インスタンス一覧
os network list             # ネットワーク一覧
os-service                  # サービス状態確認
os-restart                  # サービス再起動
```

---

## セキュリティ設定

### ファイアウォールの設定（必要に応じて）

```bash
# UFWのインストール（既にインストール済みの場合はスキップ）
sudo apt install -y ufw

# 必要なポートの開放
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS
sudo ufw allow 5000/tcp     # Keystone
sudo ufw allow 8774/tcp     # Nova
sudo ufw allow 9292/tcp     # Glance
sudo ufw allow 9696/tcp     # Neutron

# UFWの有効化（注意: SSH接続が切れないように先にSSHを許可）
sudo ufw enable
```

---

## 次のステップ

インストールが完了したら、`openstack-commands.md` を参照して以下のハンズオンを実施してください：

1. **Keystoneハンズオン**: ユーザー、プロジェクト、ロールの管理
2. **Glanceハンズオン**: イメージの管理
3. **Neutronハンズオン**: ネットワーク、サブネット、ルーターの作成
4. **Novaハンズオン**: インスタンスの起動と管理
5. **総合演習**: 完全な環境構築とインスタンスへのアクセス

---

## クイックリファレンス

### よく使うコマンド

```bash
# stackユーザーに切り替え
sudo su - stack

# 認証情報の読み込み
source ~/devstack/openrc admin admin

# サービス一覧
openstack service list

# サービスの再起動
sudo systemctl restart devstack@*

# ログの確認
tail -f /opt/stack/devstack/logs/stack.sh.log
```

### 重要なディレクトリ

```
/opt/stack/              # stackユーザーのホーム
/opt/stack/devstack/     # DevStackインストールディレクトリ
/opt/stack/logs/         # ログディレクトリ
/etc/systemd/system/     # systemdサービスファイル
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

すべてのチェックが完了したら、`openstack-commands.md` の演習を実施してください！
