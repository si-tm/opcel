# OpenStack ハンズオン コマンド集（OPCEL対策）

## 目次
1. [Docker環境でのOpenStack構築](#docker環境でのopenstack構築)
2. [環境確認](#環境確認)
3. [Keystone（認証サービス）](#keystone認証サービス)
4. [Glance（イメージサービス）](#glanceイメージサービス)
5. [Nova（コンピュートサービス）](#novaコンピュートサービス)
6. [Neutron（ネットワークサービス）](#neutronネットワークサービス)
7. [総合演習](#総合演習)

---

## Docker環境でのOpenStack構築

DockerコンテナでOpenStack環境を構築する方法です。ホストマシンを汚さずに、簡単にOpenStack環境を試すことができます。

### 🔧 Docker環境の準備

#### Dockerのディスク容量を増やす

ビルド前に以下を確認してください：

```bash
# Dockerのディスク使用状況確認
docker system df

# 不要なイメージ・コンテナ・ボリュームを削除
docker system prune -a --volumes

# Docker Desktopの場合、設定で仮想ディスクサイズを増やす
# Preferences → Resources → Disk image size を 64GB以上に設定
```

---

### 方法1: OpenStackクライアントのみ（最軽量・推奨）

既存のOpenStack環境に接続するためのクライアントのみをインストールする方法です。**ディスク容量が限られている場合はこの方法を推奨します。**

#### Dockerfile（クライアント専用・軽量版）

```dockerfile
FROM python:3.11-slim

# タイムゾーン設定
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# 最小限のパッケージのみインストール
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# OpenStackクライアントのインストール
RUN pip install --no-cache-dir \
    python-openstackclient \
    python-keystoneclient \
    python-glanceclient \
    python-novaclient \
    python-neutronclient \
    python-cinderclient

# 作業ディレクトリ
WORKDIR /workspace

# 認証情報用のテンプレート作成
RUN cat > /workspace/openrc.sh.template << 'EOF'
# OpenStack認証情報
export OS_AUTH_URL=http://your-openstack-server:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

CMD ["/bin/bash"]
```

#### 使用方法

```bash
# イメージのビルド（1GB未満）
docker build -t openstack-client -f Dockerfile.client .

# コンテナの起動
docker run -it --rm \
  --name os-client \
  -v $(pwd):/workspace \
  openstack-client /bin/bash

# コンテナ内で認証情報を設定
cp openrc.sh.template openrc.sh
vi openrc.sh  # 実際の環境情報に編集

# 認証情報の読み込み
source openrc.sh

# OpenStackコマンドの実行
openstack --version
openstack service list
openstack server list
```

---

### 方法2: DevStack（軽量版・ディスク容量節約）

学習用のOpenStack環境を構築します。**10GB以上の空き容量が必要です。**

#### Dockerfile（DevStack軽量版）

```dockerfile
FROM ubuntu:22.04

# タイムゾーン設定
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# レイヤーを分けて、不要なファイルを都度削除
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git \
    sudo \
    python3 \
    python3-pip \
    curl \
    net-tools \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# stackユーザーの作成
RUN useradd -s /bin/bash -d /opt/stack -m stack && \
    echo "stack ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# stackユーザーに切り替え
USER stack
WORKDIR /opt/stack

# DevStackのクローン（最新版のみ）
RUN git clone https://opendev.org/openstack/devstack --depth=1

# local.confの作成（最小構成）
RUN cd devstack && cat > local.conf << 'EOF'
[[local|localrc]]

# パスワード設定
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=secret
RABBIT_PASSWORD=secret
SERVICE_PASSWORD=secret

# ホストIP設定
HOST_IP=127.0.0.1

# ログ設定（ディスク節約のため最小限）
LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=False
LOG_COLOR=False

# 最小限のサービスのみ
disable_all_services
enable_service key mysql rabbit
enable_service n-api n-cpu n-cond n-sch
enable_service g-api
enable_service q-svc q-agt q-dhcp q-l3 q-meta

# イメージは手動でアップロード（自動ダウンロードを無効化）
DOWNLOAD_DEFAULT_IMAGES=False

# キャッシュディレクトリの設定
DEST=/opt/stack
DATA_DIR=$DEST/data

RECLONE=yes
EOF

# ポート公開
EXPOSE 5000 8774 9292 9696

CMD ["/bin/bash"]
```

#### 使用方法

```bash
# イメージのビルド
docker build -t openstack-devstack-lite -f Dockerfile.devstack-lite .

# コンテナの起動（特権モードが必要）
docker run -it --privileged \
  --name openstack-dev \
  -p 5000:5000 \
  -p 8774:8774 \
  -p 9292:9292 \
  -p 9696:9696 \
  -v openstack-data:/opt/stack/data \
  openstack-devstack-lite /bin/bash

# コンテナ内でDevStackをインストール
cd /opt/stack/devstack
./stack.sh

# 認証情報の読み込み
source /opt/stack/devstack/openrc admin admin

# 動作確認
openstack service list
```

---

### 方法3: Docker Compose（複数サービス管理・推奨）

#### docker-compose.yml

```yaml
version: '3.8'

services:
  # OpenStackクライアント（軽量）
  openstack-client:
    image: python:3.11-slim
    container_name: openstack-client
    volumes:
      - ./workspace:/workspace
      - ./credentials:/credentials
    working_dir: /workspace
    command: >
      bash -c "
        pip install --no-cache-dir python-openstackclient python-keystoneclient python-glanceclient python-novaclient python-neutronclient &&
        /bin/bash
      "
    stdin_open: true
    tty: true
    networks:
      - openstack-net

networks:
  openstack-net:
    driver: bridge
```

#### 使用方法

```bash
# サービスの起動
docker-compose up -d

# クライアントコンテナに入る
docker-compose exec openstack-client /bin/bash

# 認証情報の設定
cat > /workspace/openrc.sh << 'EOF'
export OS_AUTH_URL=http://your-openstack-server:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

source /workspace/openrc.sh

# OpenStackコマンドの実行
openstack server list

# サービスの停止
docker-compose down
```

---

### ディスク容量問題の解決方法

#### 問題: "No space left on device" エラー

**解決策1: Dockerのディスク容量を増やす**

```bash
# Docker Desktop（Mac/Windows）の場合
# 1. Docker Desktop を開く
# 2. Settings → Resources → Advanced
# 3. Disk image size を 64GB 以上に設定
# 4. Apply & Restart

# Linux の場合
# Docker のデータディレクトリを確認
docker info | grep "Docker Root Dir"

# ディスク容量の確認
df -h

# 不要なDockerリソースを削除
docker system prune -a --volumes
```

**解決策2: 軽量版Dockerfileを使用**

vimなどの大きなパッケージを削除：

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# 最小限のパッケージのみ（vimは除外）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git \
    sudo \
    python3 \
    python3-pip \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# 以下、同様...
```

**解決策3: マルチステージビルドを使用**

```dockerfile
# ビルドステージ
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y git python3
RUN git clone https://opendev.org/openstack/devstack --depth=1

# 実行ステージ（軽量）
FROM ubuntu:22.04
COPY --from=builder /path/to/devstack /opt/stack/devstack
# 必要な最小限のパッケージのみインストール
```

---

### トラブルシューティング

#### 1. ディスク容量不足

```bash
# 現在のディスク使用状況
docker system df -v

# 未使用のイメージを削除
docker image prune -a

# 停止中のコンテナを削除
docker container prune

# 未使用のボリュームを削除
docker volume prune

# すべて削除（注意！）
docker system prune -a --volumes
```

#### 2. ビルドが遅い

```bash
# BuildKitを使用（高速化）
export DOCKER_BUILDKIT=1
docker build -t openstack-client .

# キャッシュを使わずにビルド
docker build --no-cache -t openstack-client .
```

#### 3. メモリ不足

```bash
# Docker Desktopのメモリ設定を増やす
# Settings → Resources → Memory を 8GB 以上に設定
```

---

### 推奨する構築方法（ディスク容量別）

#### ケース1: ディスク容量が少ない（10GB未満）
→ **方法1: OpenStackクライアントのみ**
- 軽量（1GB未満）
- 外部のOpenStack環境に接続して学習

#### ケース2: ディスク容量が中程度（10-30GB）
→ **方法2: DevStack軽量版**
- 基本的なOpenStack環境を構築
- ローカルでハンズオン可能

#### ケース3: ディスク容量が十分（30GB以上）
→ **Amazon Linux 2023 on EC2**
- `setup-guide-amazon-linux.md` を参照
- 本格的な学習環境を構築

---

## 環境確認

### OpenStackクライアントのバージョン確認
```bash
openstack --version
```

### 認証情報の読み込み
```bash
source ~/devstack/openrc admin admin
```

### サービス一覧の確認
```bash
openstack service list
```

### エンドポイント一覧の確認
```bash
openstack endpoint list
```

---

## Keystone（認証サービス）

### ユーザー管理

#### ユーザー一覧表示
```bash
openstack user list
```

#### ユーザー作成
```bash
openstack user create --password <password> --email <email> <username>
```

#### ユーザー詳細表示
```bash
openstack user show <username>
```

#### ユーザー削除
```bash
openstack user delete <username>
```

### プロジェクト管理

#### プロジェクト一覧表示
```bash
openstack project list
```

#### プロジェクト作成
```bash
openstack project create --description "プロジェクトの説明" <project-name>
```

#### プロジェクト詳細表示
```bash
openstack project show <project-name>
```

#### プロジェクト削除
```bash
openstack project delete <project-name>
```

### ロール管理

#### ロール一覧表示
```bash
openstack role list
```

#### ロール作成
```bash
openstack role create <role-name>
```

#### ユーザーにロールを付与
```bash
openstack role add --user <username> --project <project-name> <role-name>
```

#### ロール割り当て一覧
```bash
openstack role assignment list
```

#### ユーザーのロール確認
```bash
openstack role assignment list --user <username> --project <project-name>
```

---

## Glance（イメージサービス）

### イメージ管理

#### イメージ一覧表示
```bash
openstack image list
```

#### イメージ詳細表示
```bash
openstack image show <image-name-or-id>
```

#### イメージアップロード（Cirrosイメージ）
```bash
# Cirrosイメージのダウンロード
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# イメージの登録
openstack image create "cirros" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

#### イメージのプロパティ設定
```bash
openstack image set --property os_type=linux <image-name-or-id>
```

#### イメージ削除
```bash
openstack image delete <image-name-or-id>
```

---

## Nova（コンピュートサービス）

### インスタンス管理

#### インスタンス一覧表示
```bash
openstack server list
```

#### インスタンス詳細表示
```bash
openstack server show <instance-name-or-id>
```

#### フレーバー一覧表示
```bash
openstack flavor list
```

#### フレーバー作成
```bash
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny
```

#### インスタンス起動
```bash
openstack server create \
  --flavor m1.tiny \
  --image cirros \
  --network private \
  <instance-name>
```

#### インスタンスの状態確認
```bash
openstack server show <instance-name> -c status
```

#### インスタンス停止
```bash
openstack server stop <instance-name>
```

#### インスタンス起動（停止中のものを再起動）
```bash
openstack server start <instance-name>
```

#### インスタンス再起動
```bash
openstack server reboot <instance-name>
```

#### インスタンス削除
```bash
openstack server delete <instance-name>
```

### コンソールアクセス

#### コンソールURL取得
```bash
openstack console url show <instance-name>
```

#### コンソールログ表示
```bash
openstack console log show <instance-name>
```

### キーペア管理

#### キーペア一覧表示
```bash
openstack keypair list
```

#### キーペア作成
```bash
openstack keypair create <keypair-name> > mykey.pem
chmod 600 mykey.pem
```

#### キーペア削除
```bash
openstack keypair delete <keypair-name>
```

### ホスト・ハイパーバイザー情報

#### コンピュートサービス一覧
```bash
openstack compute service list
```

#### ハイパーバイザー一覧
```bash
openstack hypervisor list
```

#### ハイパーバイザー詳細
```bash
openstack hypervisor show <hypervisor-id>
```

---

## Neutron（ネットワークサービス）

### ネットワーク管理

#### ネットワーク一覧表示
```bash
openstack network list
```

#### ネットワーク作成
```bash
openstack network create <network-name>
```

#### ネットワーク詳細表示
```bash
openstack network show <network-name>
```

#### ネットワーク削除
```bash
openstack network delete <network-name>
```

### サブネット管理

#### サブネット一覧表示
```bash
openstack subnet list
```

#### サブネット作成
```bash
openstack subnet create \
  --network <network-name> \
  --subnet-range 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --dns-nameserver 8.8.8.8 \
  <subnet-name>
```

#### サブネット詳細表示
```bash
openstack subnet show <subnet-name>
```

#### サブネット削除
```bash
openstack subnet delete <subnet-name>
```

### ルーター管理

#### ルーター一覧表示
```bash
openstack router list
```

#### ルーター作成
```bash
openstack router create <router-name>
```

#### 外部ネットワークへのゲートウェイ設定
```bash
openstack router set --external-gateway <external-network> <router-name>
```

#### サブネットをルーターに接続
```bash
openstack router add subnet <router-name> <subnet-name>
```

#### ルーターからサブネットを切断
```bash
openstack router remove subnet <router-name> <subnet-name>
```

#### ルーター削除
```bash
openstack router delete <router-name>
```

### セキュリティグループ管理

#### セキュリティグループ一覧
```bash
openstack security group list
```

#### セキュリティグループ作成
```bash
openstack security group create <security-group-name>
```

#### セキュリティグループルール追加（SSH許可）
```bash
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  <security-group-name>
```

#### セキュリティグループルール追加（ICMP許可）
```bash
openstack security group rule create \
  --protocol icmp \
  <security-group-name>
```

#### セキュリティグループルール一覧
```bash
openstack security group rule list <security-group-name>
```

#### セキュリティグループルール削除
```bash
openstack security group rule delete <rule-id>
```

### フローティングIP管理

#### フローティングIP一覧
```bash
openstack floating ip list
```

#### フローティングIP作成
```bash
openstack floating ip create <external-network>
```

#### フローティングIPをインスタンスに割り当て
```bash
openstack server add floating ip <instance-name> <floating-ip>
```

#### フローティングIPを解除
```bash
openstack server remove floating ip <instance-name> <floating-ip>
```

#### フローティングIP削除
```bash
openstack floating ip delete <floating-ip>
```

### ポート管理

#### ポート一覧表示
```bash
openstack port list
```

#### ポート詳細表示
```bash
openstack port show <port-id>
```

#### ポート作成
```bash
openstack port create --network <network-name> <port-name>
```

#### ポート削除
```bash
openstack port delete <port-id>
```

---

## 総合演習

### 演習1: 完全なネットワーク環境の構築

```bash
# 1. プライベートネットワークの作成
openstack network create private-network

# 2. サブネットの作成
openstack subnet create \
  --network private-network \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --dns-nameserver 8.8.8.8 \
  private-subnet

# 3. ルーターの作成
openstack router create my-router

# 4. 外部ネットワークへの接続（publicは環境により異なる）
openstack router set --external-gateway public my-router

# 5. ルーターにサブネットを接続
openstack router add subnet my-router private-subnet

# 6. セキュリティグループの作成
openstack security group create web-sg

# 7. SSHルールの追加
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  web-sg

# 8. ICMPルールの追加
openstack security group rule create \
  --protocol icmp \
  web-sg

# 9. HTTPルールの追加
openstack security group rule create \
  --protocol tcp \
  --dst-port 80 \
  web-sg
```

### 演習2: インスタンスの起動とアクセス

```bash
# 1. キーペアの作成
openstack keypair create my-keypair > my-keypair.pem
chmod 600 my-keypair.pem

# 2. インスタンスの起動
openstack server create \
  --flavor m1.small \
  --image cirros \
  --network private-network \
  --security-group web-sg \
  --key-name my-keypair \
  web-server

# 3. インスタンスの状態確認
openstack server show web-server

# 4. フローティングIPの作成
openstack floating ip create public

# 5. フローティングIPの割り当て
openstack server add floating ip web-server <floating-ip-address>

# 6. インスタンスへのSSH接続
ssh -i my-keypair.pem cirros@<floating-ip-address>
```

### 演習3: リソースのクリーンアップ

```bash
# 1. フローティングIPの解除と削除
openstack server remove floating ip web-server <floating-ip-address>
openstack floating ip delete <floating-ip-address>

# 2. インスタンスの削除
openstack server delete web-server

# 3. ルーターからサブネットを切断
openstack router remove subnet my-router private-subnet

# 4. ルーターの外部ゲートウェイ削除
openstack router unset --external-gateway my-router

# 5. ルーターの削除
openstack router delete my-router

# 6. サブネットの削除
openstack subnet delete private-subnet

# 7. ネットワークの削除
openstack network delete private-network

# 8. セキュリティグループの削除
openstack security group delete web-sg

# 9. キーペアの削除
openstack keypair delete my-keypair
```

---

## よく使うトラブルシューティングコマンド

### ログの確認
```bash
# Novaのログ
sudo journalctl -u devstack@n-* -f

# Neutronのログ
sudo journalctl -u devstack@q-* -f

# すべてのDevStackサービス
sudo systemctl status devstack@*
```

### サービスの再起動
```bash
# すべてのDevStackサービスを再起動
sudo systemctl restart devstack@*

# 特定のサービスのみ再起動
sudo systemctl restart devstack@n-api
sudo systemctl restart devstack@q-svc
```

### リソース使用状況の確認
```bash
# プロジェクトのクォータ確認
openstack quota show

# 使用状況の確認
openstack limits show --absolute
```

---

## OPCEL試験対策ポイント

### 重要コマンドの暗記
- `openstack server create`: インスタンス作成
- `openstack network create`: ネットワーク作成
- `openstack subnet create`: サブネット作成
- `openstack router create`: ルーター作成
- `openstack floating ip create`: フローティングIP作成
- `openstack security group rule create`: セキュリティルール作成

### よくある操作パターン
1. ネットワーク → サブネット → ルーター → 外部接続の順で構築
2. セキュリティグループは先に作成してからインスタンスに適用
3. フローティングIPはインスタンス起動後に割り当て
4. 削除は作成の逆順で行う（インスタンス → ネットワークリソース）

### 確認すべき出力項目
- `status`: リソースの状態（ACTIVE, DOWN, BUILD等）
- `id`: リソースの一意識別子
- `name`: リソース名
- ネットワークの場合: `provider:network_type`, `router:external`
- インスタンスの場合: `addresses`, `flavor`, `image`
