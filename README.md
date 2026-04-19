# Dify 1.0 プラグインのダウンロードと再パッケージング（オフライン化）

このツールは、Dify Marketplace や GitHub からプラグイン（.difypkg）をダウンロードし、依存する Python パッケージ（wheels）を同梱してオフライン環境でもインストール可能な形式に再パッケージングするためのものです。

---

### 1. GitHub Actions での使用方法（推奨）

最も簡単な方法です。自分の GitHub アカウントで実行できます。

1. このリポジトリを **Fork** する
2. Fork したリポジトリの **Actions** タブを開く
3. **Repackage Dify Plugin** ワークフローを選択し、**Run workflow** をクリック
   - `plugin author`: プラグインの作者名（例: `langgenius`）
   - `plugin name`: プラグイン名（例: `agent`）
   - `plugin version`: バージョン（例: `0.0.9`）
   - `Use ARM platform`: ARM版（aarch64）が必要な場合は `true`
4. 完了後、**Artifacts** セクションから生成されたファイルをダウンロードする

---

### 2. Docker での使用方法

1. **Dockerfile の CMD を編集**
   実行したいプラグインの情報に合わせて引数を変更します。
   ```dockerfile
   CMD ["./plugin_repackaging.sh", "-p", "manylinux_2_17_x86_64", "market", "antv", "visualization", "0.1.7"] 
   ```

2. **イメージのビルド**
   ```bash
   docker build -t dify-plugin-repackaging .
   ```

3. **実行（Windows の例）**
   カレントディレクトリに成果物が出力されます。
   ```cmd
   docker run -v %cd%:/app dify-plugin-repackaging
   ```

---

### 3. 前提条件と環境

- **OS**: Linux (amd64/aarch64), MacOS (x86_64/arm64)
- **依存関係**: `unzip` コマンドが必要です。
  - スクリプト内で `yum` を使用してインストールを試みますが、それ以外のディストリビューション（Ubuntu等）では、あらかじめ `sudo apt install unzip` などでインストールしておいてください。
- **Python**: Dify プラグインランタイムと合わせるため、**3.12.x** を推奨します。

#### クローン
```shell
git clone https://github.com/Tatsunobu-Eto/dify-plugin-repackaging.git
```

---

### 4. 主な機能と使い方の詳細

#### Dify Marketplace からの再パッケージング
Marketplace で公開されているプラグインを指定してオフライン化します。
```shell
./plugin_repackaging.sh market {作者名} {プラグイン名} {バージョン}
```
例: `./plugin_repackaging.sh market langgenius agent 0.0.9`

#### GitHub からの再パッケージング
GitHub の Releases に公開されている `.difypkg` を指定してオフライン化します。
```shell
./plugin_repackaging.sh github {リポジトリ名} {リリース名} {ファイル名.difypkg}
```
例: `./plugin_repackaging.sh github junjiem/dify-plugin-agent-mcp_sse 0.0.1 agent-mcp_see.difypkg`

#### 手元の .difypkg ファイルを再パッケージング
すでにローカルにあるファイルを展開し、依存関係を同梱します。
```shell
./plugin_repackaging.sh local ./db_query.difypkg
```

#### クロスプラットフォーム対応（`-p` オプション）
実行環境と異なるプラットフォーム（OS/CPU）向けに作成する場合、`-p` で pip のプラットフォーム文字列を指定します。
- **x86_64/amd64**: `manylinux2014_x86_64` または `manylinux_2_17_x86_64`
- **arm64/aarch64**: `manylinux2014_aarch64` または `manylinux_2_17_aarch64`

---

### 5. Dify プラットフォーム側の設定（重要）

自作や未審査のプラグインをインストールしたり、大容量のパッケージを扱うために Dify の `.env` を更新する必要があります。

- **署名検証の解除**: `FORCE_VERIFYING_SIGNATURE=false`
  - これにより、Marketplace 未掲載のカスタムプラグインのインストールが可能になります。
- **最大サイズ制限の拡張**: `PLUGIN_MAX_PACKAGE_SIZE=524288000` (500MB)
- **Nginx のアップロード制限**: `NGINX_CLIENT_MAX_BODY_SIZE=500M`

---

### 6. プラグインのインストール

Dify のプラグイン管理ページから「ローカルパッケージファイル（Local Package File）」を選択し、生成された `*-offline.difypkg` をアップロードして完了です。

![install_plugin_via_local](./images/install_plugin_via_local.png)

---

### ライセンス・履歴
[Star History Chart](https://star-history.com/#junjiem/dify-plugin-repackaging&Date)
