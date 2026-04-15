# Ubuntu 操作マニュアル（Macユーザー向け）

このドキュメントでは、MacユーザーがUbuntuを使う際の基本的な操作方法とMacとの違いを説明します。

## 1. ターミナル

### 1-1. ターミナルの起動

| 方法 | 操作 |
|---|---|
| GUI | Ctrl+Alt+T、またはアプリ一覧から「Terminal」 |
| ショートカット | Ctrl+Alt+T |

### 1-2. シェル

Ubuntuのデフォルトシェルは **bash** です。Macのzshと異なり、プロンプトは以下のようになります:

```bash
user@hostname:~$
```

- `~` = ホームディレクトリ（`/home/user`）
- `$` = 一般ユーザー（rootは `#`）

## 2. パッケージ管理

### 2-1. Mac (Homebrew) vs Ubuntu (apt)

| 操作 | Mac (Homebrew) | Ubuntu (apt) |
|---|---|---|
| パッケージの検索 | `brew search <name>` | `apt search <name>` |
| パッケージのインストール | `brew install <name>` | `sudo apt install <name>` |
| パッケージのアンインストール | `brew uninstall <name>` | `sudo apt remove <name>` |
| パッケージ一覧 | `brew list` | `dpkg -l` または `apt list --installed` |
| パッケージの更新 | `brew upgrade` | `sudo apt upgrade` |
| パッケージ情報の更新 | `brew update` | `sudo apt update` |

### 2-2. aptの基本操作

```bash
# パッケージ情報の更新（インストール前に必ず実行）
sudo apt update

# パッケージのインストール
sudo apt install <package_name>

# 複数パッケージのインストール
sudo apt install git curl wget vim

# パッケージの削除（設定ファイルは残す）
sudo apt remove <package_name>

# パッケージの完全削除（設定ファイルも消す）
sudo apt purge <package_name>

# 不要なパッケージの自動削除
sudo apt autoremove
```

### 2-3. Node.js / npm / bun

```bash
# bun（推奨）
curl -fsSL https://bun.sh/install | bash

# Node.js (n経由)
sudo apt install -y nodejs npm

# グローバルパッケージ
npm install -g <package>
```

## 3. ファイルシステム

### 3-1. ディレクトリ構造

| Mac | Ubuntu | 説明 |
|---|---|---|
| `/Users/<user>` | `/home/<user>` | ホームディレクトリ |
| `/Applications` | `/usr/bin`, `/usr/share/applications` | アプリ |
| `/Library` | `/usr/lib`, `/usr/share` | システムライブラリ |
| `/System` | `/` | システムルート |
| `/tmp` | `/tmp` | 一時ファイル |

### 3-2. パスの表記

| Mac | Ubuntu | 例 |
|---|---|---|
| `~/Documents` | `~/Documents` または `~/ドキュメント` | ドキュメント |
| `/Users/user` | `/home/user` または `~` | ホーム |

### 3-3. 権限

```bash
# ファイルの所有者と権限を確認
ls -la filename

# 所有者変更
sudo chown user:group filename

# 権限変更
chmod 755 filename  # rwxr-xr-x（実行可能）
chmod 644 filename  # rw-r--r--（通常ファイル）
chmod 600 filename  # rw-------（秘密鍵等）

# ディレクトリの権限を再帰的に変更
chmod -R 755 directory/
```

## 4. ディレクトリ操作

```bash
# 現在のディレクトリを確認
pwd

# ディレクトリの中身を表示
ls
ls -la    # 詳細（隠しファイル含む）
ls -lh    # サイズを読みやすく表示

# ディレクトリの移動
cd /path/to/directory
cd ~      # ホームへ
cd ..     # 一つ上へ

# ディレクトリの作成
mkdir newdir
mkdir -p a/b/c  # 階層ディレクトリを一気に作成

# ファイル/ディレクトリのコピー
cp src.txt dest.txt
cp -r srcdir/ destdir/   # ディレクトリ全体をコピー

# ファイル/ディレクトリの移動・リネーム
mv oldname newname

# ファイル/ディレクトリの削除
rm filename
rm -r directory/        # ディレクトリ全体を削除（確認なし）
rm -ri directory/       # ディレクトリ全体を削除（確認あり）
```

## 5. テキストエディタ

### 5-1. nano（標準搭載）

```bash
nano filename.txt
```

| 操作 | ショートカット |
|---|---|
| 保存 | Ctrl+O → Enter |
| 終了 | Ctrl+X |
| 検索 | Ctrl+W |
| 置換 | Ctrl+\ |
| 行番号を表示 | Alt+R → 行番号を入力 → Enter |
| ヘルプ | Ctrl+G |

### 5-2. vim（インストール済み）

```bash
vim filename.txt
```

| モード | 操作 | 入力 |
|---|---|---|
| ノurzモード | カーソル移動 | `h` `j` `k` `l` |
| ノurzモード | 終了 | `:q` |
| ノurzモード | 強制終了 | `:q!` |
| ノurzモード | 保存して終了 | `:wq` |
| ノurzモード | 編集モードへ | `i` または `a` |
| 編集モード | ノurzモードへ | `Esc` |

## 6. プロセス管理

### 6-1. プロセスの確認

```bash
# 実行中のプロセス一覧
ps aux

# 特定のプロセスを検索
ps aux | grep node

# ポートを使用しているプロセス
ss -tlnp | grep :3000

# ポートを使用しているPIDを特定
fuser 3000/tcp
```

### 6-2. プロセスの終了

```bash
# 通常終了
kill <PID>

# 強制終了（応答がない場合）
kill -9 <PID>

# 特定ポートのプロセスを終了
fuser -k 3000/tcp

# 名前でプロセスを終了
killall node
pkill -f "next dev"
```

### 6-3. サービス管理（systemctl）

```bash
# Dockerの例
sudo systemctl start docker     # 起動
sudo systemctl stop docker      # 停止
sudo systemctl restart docker   # 再起動
sudo systemctl enable docker    # 自動起動を有効化
sudo systemctl status docker    # 状態確認
```

## 7. ネットワーク

```bash
# IPアドレスの確認
ip addr show
# または
hostname -I

# ポートの確認
ss -tlnp
# 特定ポート: ss -tlnp | grep :3000

# 接続テスト
curl -sS http://localhost:3000
curl -I https://example.com

# DNS確認
nslookup example.com
dig example.com

# ルーティング確認
ip route
```

## 8. ユーザーと権限

### 8-1. sudo

```bash
# root権限でコマンドを実行
sudo <command>

# rootユーザーになる（推奨されません）
sudo -i

# 環境変数を引き継いでrootで実行
sudo -E <command>
```

### 8-2. dockerグループへの追加

```bash
sudo usermod -aG docker $USER
# 反映にはログアウト→再ログインが必要
# または即時反映:
newgrp docker
```

## 9. Macとの違い まとめ

| 項目 | Mac | Ubuntu |
|---|---|---|
| パッケージマネージャー | Homebrew | apt |
| パッケージプレフィックス | `brew` | `sudo apt` |
| アプリ終了 | Cmd+Q | `kill` / `xkill` |
| アプリ強制終了 | Cmd+Option+Esc | `kill -9 <PID>` |
| スクリーンショット | Cmd+Shift+3/4 | PrtScn / `gnome-screenshot` |
| クリップボード | Cmd+C / Cmd+V | Ctrl+Shift+C / Ctrl+Shift+V |
| Spotlight検索 | Cmd+Space | Super（Windowsキー） |
| 設定 | システム環境設定 | 「設定」アプリ / `gnome-control-center` |
| ファイルマネージャー | Finder | Nautilus（「ファイル」） |
| パス区切り | `/` | `/`（同じ） |
| 大文字小文字 | 区別する | 区別する（同じ） |
| ファイルシステム | APFS（大文字小文字区別） | ext4（大文字小文字区別） |

## 10. よく使うコマンド早見表

```bash
# システム情報
uname -a           # カーネル情報
lsb_release -a     # Ubuntuバージョン
df -h               # ディスク容量
free -h             # メモリ容量
uptime              # 稼動時間
whoami              # 現在のユーザー

# ファイル操作
ls -la              # ファイル一覧（詳細）
find . -name "*.ts"  # ファイル検索
grep -r "pattern" . # 内容検索
wc -l filename      # 行数カウント
head -n 20 filename # 先頭20行
tail -f filename    # リアルタイム表示

# テキスト処理
cat filename        # ファイル内容表示
less filename       # ページャーで表示（qで終了）
sort filename       # 行のソート
uniq                # 重複行の削除
sed 's/old/new/g'   # 文字列置換
awk '{print $1}'     # フィールド抽出

# アーカイブ
tar czf archive.tar.gz directory/   # 圧縮
tar xzf archive.tar.gz              # 展開
zip -r archive.zip directory/        # ZIP圧縮
unzip archive.zip                   # ZIP展開
```
