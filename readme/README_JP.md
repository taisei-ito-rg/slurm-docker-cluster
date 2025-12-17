# Slurm Docker Cluster

<p align="center">
    <b> <a href="../README.md">English</a> | <a href="./README_CN.md">简体中文</a> | 日本語 </b>
</p>

**Slurm Docker Cluster**は、Docker Composeを使用して迅速にデプロイできるマルチコンテナSlurmクラスタです。このリポジトリは、開発、テスト、または軽量な使用のための堅牢なSlurm環境のセットアッププロセスを簡素化します。

## 🏁 はじめに

DockerでSlurmを起動して実行するには、以下のツールがインストールされていることを確認してください：

- **[Docker](https://docs.docker.com/get-docker/)**
- **[Docker Compose](https://docs.docker.com/compose/install/)**

リポジトリをクローンします：

```bash
git clone https://github.com/giovtorres/slurm-docker-cluster.git
cd slurm-docker-cluster
```

## 🔢 Slurmバージョンの選択

このプロジェクトは複数のSlurmバージョンをサポートしています。バージョンを選択するには、`.env.example`を`.env`にコピーして`SLURM_VERSION`を設定します：

```bash
cp .env.example .env
# .envを編集して設定:
SLURM_VERSION=25.05.3   # 最新安定版（デフォルト）
# または:
SLURM_VERSION=24.11.6   # 以前の安定版リリース
```

**サポートされているバージョン:** 25.05.x, 24.11.x

## 🏗️ アーキテクチャサポート

このプロジェクトは**AMD64 (x86_64)**と**ARM64 (aarch64)**の両方のアーキテクチャをサポートしています。ビルドシステムは自動的にアーキテクチャを検出します。特別な設定は不要です - 単にビルドして実行してください：

```bash
make build
make up
```

## 🚀 クイックスタート（Makeを使用）

最も簡単な開始方法は、提供されているMakefileを使用することです：

```bash
# クラスタをビルドして起動
make up

# すべてが正常に動作することを確認するためにテストを実行
make test

# クラスタのステータスを表示
make status
```

利用可能なすべてのコマンドを確認：
```bash
make help
```

## 📦 コンテナとボリューム

このセットアップは以下のコンテナで構成されています：

- **mysql**: ジョブとクラスタのデータを保存します。
- **slurmdbd**: Slurmデータベースを管理します。
- **slurmctld**: ジョブとリソース管理を担当するSlurmコントローラ。
- **slurmrestd**: クラスタへのHTTP/JSONアクセス用のREST APIデーモン。
- **c1, c2**: 計算ノード（`slurmd`を実行）。

### 永続ボリューム：

- `etc_munge`: `/etc/munge`にマウント - 認証キー
- `etc_slurm`: `/etc/slurm`にマウント - 設定ファイル（ライブ編集可能）
- `slurm_jobdir`: `/data`にマウント - すべてのノードで共有されるジョブファイル
- `var_lib_mysql`: `/var/lib/mysql`にマウント - データベースの永続化
- `var_log_slurm`: `/var/log/slurm`にマウント - ログファイル

## 🛠️ クラスタのビルドと起動

### ビルド

クラスタをビルドして起動する最も簡単な方法はMakeを使用することです：

```bash
# デフォルトバージョン（25.05.3）でイメージをビルド
make build

# または1つのコマンドでビルドして起動
make up
```

別のバージョンをビルドするには、`.env`の`SLURM_VERSION`を更新します：

```bash
make set-version VER=24.11.6

# ビルド
make build
```

または、Docker Composeを直接使用します：

```bash
docker compose build
```

### 起動

デタッチモードでクラスタを起動：

```bash
make up
```

クラスタのステータスを確認：

```bash
make status
```

ログを表示：

```bash
make logs
```

> **注意**: クラスタは初回起動時にSlurmDBDに自動的に登録されます。すべてのサービスが正常になり自動登録されるまで、起動後約15〜20秒待ってください。

## 🖥️ クラスタの使用

### コントローラへのアクセス

Slurmコントローラでシェルを開きます：

```bash
make shell
# または: docker exec -it slurmctld bash
```

クラスタのステータスを確認：

```bash
[root@slurmctld /]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up   infinite      2   idle c[1-2]
```

### ジョブの投入

`/data`ディレクトリはジョブファイル用にすべてのノードで共有されています：

```bash
[root@slurmctld /]# cd /data/
[root@slurmctld data]# sbatch --wrap="hostname"
Submitted batch job 2
[root@slurmctld data]# cat slurm-2.out
c1
```

### サンプルジョブの実行

付属のサンプルスクリプトを使用：

```bash
make run-examples
```

これにより、シンプルなホスト名テスト、CPU集約的なワークロード、マルチノードジョブなどのサンプルジョブが実行されます。

## 🔄 クラスタの管理

クラスタを停止（データは保持）：

```bash
make down
```

クラスタを再起動：

```bash
make up
```

完全なクリーンアップ（すべてのデータとボリュームを削除）：

```bash
make clean
```

設定の更新、バージョンの切り替え、テストなどのその他のワークフローについては、以下の**一般的なワークフロー**セクションを参照してください。

## ⚙️ 高度な設定

### マルチアーキテクチャビルド

クロスプラットフォームビルドまたは明示的なアーキテクチャ選択（`arm64`または`amd64`）には、Docker Buildxを使用します：

```bash
docker buildx build \
  --platform linux/arm64 \
  --build-arg SLURM_VERSION=25.05.3 \
  --build-arg TARGETARCH=arm64 \
  --load \
  -t slurm-docker-cluster:25.05.3 \
  .
```

**注意**: クロスプラットフォームビルドはQEMUエミュレーションを使用するため、ネイティブビルドよりも遅い場合があります。

### ライブ設定更新

`etc_slurm`ボリュームがマウントされているため、再ビルドせずに設定を変更できます：

**方法1 - 直接編集（再起動後も永続化）:**
```bash
docker exec -it slurmctld vi /etc/slurm/slurm.conf
make reload-slurm
```

**方法2 - config/ディレクトリから変更をプッシュ:**
```bash
# config/25.05/またはconfig/common/の設定ファイルをローカルで編集
vi config/25.05/slurm.conf

# コンテナにプッシュ（.envからバージョンを自動検出）
make update-slurm FILES="slurm.conf"

# または複数のファイルを更新
make update-slurm FILES="slurm.conf slurmdbd.conf"
```

**方法3 - 新しい設定でイメージを再ビルド:**
```bash
# 恒久的な変更の場合
vi config/25.05/slurm.conf
make rebuild
```

これにより、ノードの追加/削除や新しい設定のテストを動的に簡単に行うことができます。

## 📖 一般的なワークフロー

### Makeの使用（推奨）

#### 初回セットアップ:
```bash
# クラスタをビルドして起動
make up

# すべてが正常に動作していることを確認
make test

# クラスタのステータスを確認
make status
```

#### 日常的な開発:
```bash
# ログを表示
make logs

# コントローラでシェルを開く
make shell

# シェル内:
cd /data
sbatch --wrap="hostname"
squeue
```

#### 変更のテスト:
```bash
# 設定ファイルを編集した後
make down
make start
make test
```

#### クリーンアップ:
```bash
# クラスタを停止（データは保持）
make down

# 完全なクリーンアップ（すべてのデータを削除）
make clean
```

### 例: テストジョブの実行

```bash
# クラスタを起動
make start

# サンプルジョブをクラスタにコピー
docker cp examples/jobs slurmctld:/data/

# シンプルなジョブを投入
docker exec slurmctld bash -c "cd /data/jobs && sbatch simple_hostname.sh"

# マルチノードジョブを投入
docker exec slurmctld bash -c "cd /data/jobs && sbatch multi_node.sh"

# ジョブキューを監視
docker exec slurmctld squeue

# ジョブの出力を表示
docker exec slurmctld bash -c "ls -lh /data/jobs/*.out"
docker exec slurmctld bash -c "cat /data/jobs/hostname_test_*.out"
```

### 例: 異なるSlurmバージョンのテスト

```bash
# 現在のバージョンを確認
make version

# サポートされているすべてのバージョンをビルド
make build-all

# 特定のバージョンをテスト
make test-version VER=24.11.6

# すべてのバージョンをテスト（包括的）
make test-all

# 別のバージョンに切り替えて使用
make set-version VER=24.11.6
make rebuild
make test
```

### 例: 開発ワークフロー

```bash
# 朝: クラスタを起動
make start

# 機能の開発、ローカルテスト
make test

# 問題が発生した場合はログを確認
make logs

# 夕方: クラスタを停止
make down

# 翌日: クイック再起動
make start
```

### Makefileコマンドリファレンス

| コマンド | 説明 |
|---------|-------------|
| `make help` | 利用可能なすべてのコマンドを表示 |
| `make build` | Dockerイメージをビルド |
| `make up` | コンテナを起動 |
| `make down` | コンテナを停止 |
| `make clean` | コンテナとボリュームを削除 |
| `make logs` | コンテナのログを表示 |
| `make test` | テストスイートを実行 |
| `make status` | クラスタのステータスを表示 |
| `make shell` | slurmctldでシェルを開く |
| `make update-slurm FILES="..."` | config/ディレクトリから設定ファイルを更新 |
| `make reload-slurm` | 再起動せずにSlurm設定をリロード |
| **マルチバージョンコマンド** | |
| `make version` | 現在のSlurmバージョンを表示 |
| `make set-version VER=24.11.6` | .envでSlurmバージョンを設定 |
| `make build-all` | サポートされているすべてのバージョンをビルド |
| `make test-version VER=24.11.6` | 特定のバージョンをテスト |
| `make test-all` | すべてのバージョンをテスト |

## 🤝 コントリビューション

コミュニティからのコントリビューションを歓迎します！機能の追加、バグの修正、ドキュメントの改善を行いたい場合は：

1. このリポジトリをフォークします。
2. 新しいブランチを作成します：`git checkout -b feature/your-feature`
3. プルリクエストを提出します。

## 📄 ライセンス

このプロジェクトは[MITライセンス](../LICENSE)の下でライセンスされています。
