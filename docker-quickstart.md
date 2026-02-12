# OpenStack Docker環境 クイックスタートガイド

## 問題: コンテナが起動直後に停止する

コンテナが `-d` (デタッチモード) で起動すると、すぐに停止してしまう問題の解決方法です。

---

## 解決方法1: 正しいDockerfile（推奨）

### Dockerfile.client（軽量・安定版）

```dockerfile
FROM python:3.11-slim

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# 最小限のパッケージ
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    vim-tiny \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# OpenStackクライアントのインストール
RUN pip install --no-cache-dir \
    python-openstackclient \
    python-keystoneclient \
    python-glanceclient \
    python-novaclient \
    python-neutronclient

WORKDIR /workspace

# 認証情報テンプレート
RUN cat > /workspace/openrc.sh.template << 'EOF'
export OS_AUTH_URL=http://your-openstack:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

# コンテナを起動したままにする
CMD ["tail", "-f", "/dev/null"]
```

### ビルドと起動

```bash
# イメージのビルド
docker build -t openstack-client -f Dockerfile.client .

# コンテナの起動
docker run -d \
  --name openstack-client \
  -v $(pwd):/workspace \
  openstack-client

# コンテナに入る
docker exec -it openstack-client /bin/bash

# 動作確認
openstack --version
```

---

## 解決方法2: 既存コンテナの確認とクリーンアップ

### コンテナの状態確認

```bash
# すべてのコンテナを表示
docker ps -a

# 特定のコンテナのログを確認
docker logs 45cbdb0d37c0

# コンテナの詳細情報
docker inspect 45cbdb0d37c0
```

### クリーンアップと再起動

```bash
# 停止したコンテナを削除
docker rm -f 45cbdb0d37c0

# または、名前で削除
docker rm -f openstack-dev

# すべての停止コンテナを削除
docker container prune -f

# イメージも削除して再ビルド
docker rmi openstack-kolla
```

---

## 解決方法3: インタラクティブモードで起動

デタッチモード（-d）の代わりに、インタラクティブモードで起動：

```bash
# -itフラグで起動（-d は使わない）
docker run -it --privileged \
  --name openstack-dev \
  -p 5000:5000 \
  -p 8774:8774 \
  -p 9292:9292 \
  -p 9696:9696 \
  openstack-client \
  /bin/bash
```

または、コンテナを起動したままにする：

```bash
# tail -f でコンテナを起動したままにする
docker run -d --privileged \
  --name openstack-dev \
  -p 5000:5000 \
  -p 8774:8774 \
  -p 9292:9292 \
  -p 9696:9696 \
  openstack-client \
  tail -f /dev/null

# 別のターミナルからコンテナに入る
docker exec -it openstack-dev /bin/bash
```

---

## EC2上での実践的な手順（Amazon Linux 2023）

EC2インスタンス上でDockerを使ってOpenStackクライアントを起動する完全な手順：

### 1. Dockerのインストール（Amazon Linux 2023）

```bash
# Dockerのインストール
sudo dnf install -y docker

# Dockerサービスの起動
sudo systemctl start docker
sudo systemctl enable docker

# 現在のユーザーをdockerグループに追加
sudo usermod -aG docker $USER

# 再ログイン（グループ変更を反映）
exit
# 再度SSHでログイン
```

### 2. Dockerfileの作成

```bash
# 作業ディレクトリの作成
mkdir -p ~/openstack-docker
cd ~/openstack-docker

# Dockerfileの作成
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir \
    python-openstackclient \
    python-keystoneclient \
    python-glanceclient \
    python-novaclient \
    python-neutronclient

WORKDIR /workspace

CMD ["tail", "-f", "/dev/null"]
EOF
```

### 3. イメージのビルドとコンテナの起動

```bash
# イメージのビルド
docker build -t openstack-client .

# コンテナの起動
docker run -d \
  --name openstack-client \
  -v $(pwd):/workspace \
  openstack-client

# コンテナの状態確認
docker ps

# コンテナに入る
docker exec -it openstack-client /bin/bash
```

### 4. コンテナ内でのOpenStack設定

```bash
# コンテナ内で実行
cd /workspace

# 認証情報ファイルの作成
cat > openrc.sh << 'EOF'
export OS_AUTH_URL=http://your-openstack-server:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

# 認証情報の読み込み
source openrc.sh

# 動作確認
openstack --version

# OpenStack環境に接続（実際のサーバーが必要）
# openstack service list
```

---

## トラブルシューティング

### 問題1: コンテナがすぐに停止する

**原因**: Dockerfileの `CMD` または `ENTRYPOINT` が指定されていない、または即座に終了するコマンド

**解決策**:
```dockerfile
# Dockerfileの最後に追加
CMD ["tail", "-f", "/dev/null"]
```

または起動時に指定：
```bash
docker run -d openstack-client tail -f /dev/null
```

### 問題2: コンテナが存在しない

**原因**: コンテナ名が間違っている、または削除された

**解決策**:
```bash
# すべてのコンテナを確認
docker ps -a

# 正しいコンテナIDまたは名前を使用
docker exec -it <正しいコンテナ名またはID> /bin/bash
```

### 問題3: 権限エラー

**原因**: dockerグループに所属していない

**解決策**:
```bash
sudo usermod -aG docker $USER
# ログアウト→ログインで反映

# または一時的にsudoを使用
sudo docker exec -it openstack-client /bin/bash
```

### 問題4: ポートが既に使用されている

**原因**: 既に同じポートを使用しているコンテナがある

**解決策**:
```bash
# 使用中のポートを確認
sudo netstat -tulpn | grep -E ':(5000|8774|9292|9696)'

# または、ポート指定を変更
docker run -d \
  --name openstack-client \
  -p 15000:5000 \  # ホスト側を15000に変更
  openstack-client
```

---

## Docker Composeを使った簡単な管理（推奨）

### docker-compose.yml

```yaml
version: '3.8'

services:
  openstack-client:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: openstack-client
    volumes:
      - ./workspace:/workspace
    stdin_open: true
    tty: true
    command: tail -f /dev/null
```

### 使用方法

```bash
# サービスの起動
docker-compose up -d

# コンテナに入る
docker-compose exec openstack-client /bin/bash

# ログの確認
docker-compose logs -f

# サービスの停止
docker-compose down
```

---

## まとめ: 推奨する手順

### 最もシンプルな方法（初心者向け）

```bash
# 1. Dockerfileを作成
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
RUN pip install --no-cache-dir python-openstackclient
WORKDIR /workspace
CMD ["tail", "-f", "/dev/null"]
EOF

# 2. ビルド
docker build -t openstack-client .

# 3. 起動
docker run -d --name openstack-client openstack-client

# 4. コンテナに入る
docker exec -it openstack-client /bin/bash

# 5. 動作確認
openstack --version
```

この手順なら、確実にコンテナが起動したままになります！
