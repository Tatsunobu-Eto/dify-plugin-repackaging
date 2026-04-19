# Dify プラグイン オフライン化ガイド（自分用）

> **目的**: ネットワーク制限のある環境でも Dify プラグインをインストールできるよう、
> Python 依存パッケージを `.difypkg` に同梱した「オフライン版パッケージ」を作成する。

---

## ⚡ TL;DR（毎回やること）

### 方法A: GitHub Actions（ネット環境あり、一番楽）

1. GitHub でこのリポジトリを開く
2. **Actions** タブ → **Repackage Dify Plugin** → **Run workflow**
3. 入力フォームに記入して実行

   | フィールド | 説明 | 例 |
   |---|---|---|
   | `plugin author` | プラグイン作者名 | `langgenius` |
   | `plugin name` | プラグイン名 | `agent` |
   | `plugin version` | バージョン | `0.0.9` |
   | `Use ARM platform` | ARM環境向けなら `true` | `false`（通常はfalse） |

4. 完了後 **Artifacts** セクションから `*-offline.difypkg` をダウンロード

---

### 方法B: Docker（ローカルで完結させたい場合）

```cmd
# 1. Dockerfile の CMD を書き換える
#    [market] [作者名] [プラグイン名] [バージョン]
#    例: langgenius / agent / 0.0.9 の場合
CMD ["./plugin_repackaging.sh", "-p", "manylinux_2_17_x86_64", "market", "langgenius", "agent", "0.0.9"]

# 2. ビルド
docker build -t dify-plugin-repackaging .

# 3. 実行（Windows）
docker run -v %cd%:/app dify-plugin-repackaging
```

#### CMDを書き換えずに実行する場合（上書き実行）
```cmd
docker run -v %cd%:/app dify-plugin-repackaging ^
  ./plugin_repackaging.sh -p manylinux_2_17_x86_64 market langgenius agent 0.0.9
```

---

### 方法C: ローカル直接実行（Linux/Mac のみ）

```bash
./plugin_repackaging.sh -p manylinux_2_17_x86_64 market langgenius agent 0.0.9
```

---

## 🔍 プラグイン情報の調べ方

### Dify Marketplace からのプラグインの場合

Marketplace の URL や表示から以下を特定する：

```
https://marketplace.dify.ai/plugins/{作者名}/{プラグイン名}
                                          ↑           ↑
                                    plugin_author  plugin_name
```

バージョンは Marketplace のプラグイン詳細ページで確認。

**使用コマンド例:**
```bash
./plugin_repackaging.sh market {作者名} {プラグイン名} {バージョン}
# 例:
./plugin_repackaging.sh market langgenius agent 0.0.9
./plugin_repackaging.sh market antv visualization 0.1.7
./plugin_repackaging.sh market junjiem mcp_sse 0.0.1
```

### GitHub のプラグインの場合

GitHub Releases ページからファイル名を確認する。

```
https://github.com/{org/repo}/releases/tag/{タグ名}
                                               ↑
                                        Release title
```

**使用コマンド例:**
```bash
./plugin_repackaging.sh github {org/repo} {タグ名} {アセットファイル名.difypkg}
# 例:
./plugin_repackaging.sh github junjiem/dify-plugin-tools-dbquery v0.0.2 db_query.difypkg
./plugin_repackaging.sh github junjiem/dify-plugin-agent-mcp_sse 0.0.1 agent-mcp_see.difypkg
```

### 手元に .difypkg がある場合

```bash
./plugin_repackaging.sh local ./プラグイン名.difypkg
# 例:
./plugin_repackaging.sh local ./my_plugin.difypkg
```

---

## 🎛️ オプション一覧

```
./plugin_repackaging.sh [-p platform] [-s suffix] [-R] {market|github|local} ...

  -p platform   対象OS/アーキのwheelをダウンロード（クロスコンパイル用）
  -s suffix     出力ファイルの末尾名（デフォルト: offline）
  -R            プレリリースバージョンを許可（uv --prerelease=allow）
```

### `-p` プラットフォーム 早見表

| 実行環境 | `-p` に指定する値 |
|---|---|
| Linux x86_64 / amd64（一般的なサーバー） | `manylinux_2_17_x86_64` |
| Linux ARM64 / aarch64（Raspberry Pi 等） | `manylinux_2_17_aarch64` |
| 旧形式 x86_64 | `manylinux2014_x86_64` |
| 旧形式 ARM64 | `manylinux2014_aarch64` |

> **迷ったら `manylinux_2_17_x86_64` を選ぶ（最も一般的）**

---

## 📁 出力ファイルについて

| ファイル名 | 説明 |
|---|---|
| `{作者}-{プラグイン}_{バージョン}.difypkg` | ダウンロードしたオリジナル |
| `{作者}-{プラグイン}_{バージョン}-offline.difypkg` | ★ これを使う（wheel同梱済み） |
| `{作者}-{プラグイン}_{バージョン}/` | 展開された中身（作業ディレクトリ） |

---

## 🖥️ Dify プラットフォームへのインストール

### 1. Dify 側の設定変更（初回のみ）

Dify のサーバー上 `.env` ファイルを編集：

```env
# 署名なしプラグインのインストールを許可
FORCE_VERIFYING_SIGNATURE=false

# 最大パッケージサイズを500MBに拡張
PLUGIN_MAX_PACKAGE_SIZE=524288000

# Nginxのアップロード上限を500MBに拡張
NGINX_CLIENT_MAX_BODY_SIZE=500M
```

### 2. インストール手順

1. Dify 管理画面 → **プラグイン管理**
2. **ローカルパッケージファイルから**を選択
3. `*-offline.difypkg` をアップロード

---

## 🚨 トラブルシューティング

### `unzip: command not found`
```bash
# Ubuntu/Debian
apt-get install unzip
# CentOS/RHEL
yum install unzip
```

### `pip download` が失敗する
- `-p` で指定したプラットフォームのwheelが存在しない可能性あり
- `--only-binary=:all:` の制約で source distribution しかない場合がある
- 一時的にプラットフォーム指定を外して試す

### `uv lock failed`
- `pyproject.toml` の依存関係に問題がある可能性
- `-R` オプション（プレリリース許可）を追加して再試行
  ```bash
  ./plugin_repackaging.sh -R -p manylinux_2_17_x86_64 market {作者} {名前} {バージョン}
  ```

### GitHub Actions でArtifactが見つからない
- ワークフロー実行ログを確認
- プラグイン名・バージョンのスペルミスを確認（大文字小文字に注意）

---

## 🔗 参考リンク

- [Dify Marketplace](https://marketplace.dify.ai)
- [このリポジトリ（junjiem/dify-plugin-repackaging）](https://github.com/junjiem/dify-plugin-repackaging)
- [Dify 公式ドキュメント](https://docs.dify.ai)

---

## 📝 実施ログ（今まで変換したプラグイン）

| 日付 | プラグイン | バージョン | コマンド/方法 | 備考 |
|---|---|---|---|---|
| 2026-04-19 | stvlynn/lmstudio | 0.0.2 | Docker, market | サンプルとして同梱済み |

> 変換するたびにこの表に追記しておくと次回の参考になる。
