# Ubuntu 24.04 LTS インストールマニュアル

このドキュメントでは、Ubuntu 24.04 LTSのインストールから初期設定までの手順を説明します。

## 1. インストールメディアの作成

### 1-1. ISOイメージのダウンロード

https://ubuntu.com/download/desktop から Ubuntu 24.04 LTS Desktop のISOイメージをダウンロードします。

- **推奨**: Ubuntu 24.04.x LTS（Long Term Support）
- **サイズ**: 約5.5 GB
- **言語**: 日本語 Remix または英語版（どちらでもインストール後に日本語化可能）

### 1-2. USBメモリの準備

**必要なもの:**
- 8 GB以上のUSBメモリ（中身は消えます）

**Rufus（Windows）の場合:**
1. https://rufus.ie から Rufus をダウンロード
2. USBメモリを挿入
3. Rufusを起動し、以下を設定:
   - デバイス: USBメモリを選択
   - ブートの選択: イメージを選択 → ISOファイルを指定
   - パーティション方式: GPT
   - ターゲットシステム: UEFI（非CSM）
4. 「開始」をクリック

**balenaEtcher（Mac/Linux）の場合:**
1. https://etcher.balena.io からダウンロード
2. USBメモリを挿入
3. ISOファイルを選択し、書き込み

## 2. Ubuntuのインストール

### 2-1. USBから起動

1. USBメモリをPCに挿入
2. PCの電源を入れ、起動時にBIOS/UEFI設定画面に入る
   - **一般的なキー**: F2, F12, Del, Esc（メーカーによる）
3. ブート順序を変更し、USBメモリを優先
4. 「Install Ubuntu」を選択

### 2-2. インストーラの設定

1. **言語**: English または 日本語
2. **キーボードレイアウト**: Japanese
3. **インストール種別**: Normal installation（推奨）
4. **インストールタイプ**:
   - ディスク全体を消去（推奨、他OSと共存しない場合）
   - または「Advanced features」→「LVM」→「Encrypt」を無効化
5. **タイムゾーン**: Asia/Tokyo
6. **ユーザー情報**: ユーザー名とパスワードを設定
7. 「Install Now」をクリック

### 2-3. インストール完了

1. 「Installation Complete」と表示されたら再起動
2. USBメモリを抜く（指示が出たら）
3. Ubuntuが起動することを確認

## 3. 初期セットアップ

### 3-1. ウェルカム画面

初回起動時に設定ウィザードが表示されます:

1. **Connect to Online Accounts**: スキップ（後で設定可能）
2. **Livepatch**: スキップ（後で設定可能）
3. **Help improve Ubuntu**: スキップまたはオプトアウト

### 3-2. 日本語化（英語版をインストールした場合）

```bash
# 日本語パッケージのインストール
sudo apt update
sudo apt install -y language-pack-ja fonts-noto-cjk

# システム言語を日本語に設定
sudo update-locale LANG=ja_JP.UTF-8

# 確認
locale
```

### 3-3. タイムゾーンの確認

```bash
timedatectl
# Asia/Tokyo (JST, +0900) であることを確認
```

異なる場合は:
```bash
sudo timedatectl set-timezone Asia/Tokyo
```

### 3-4. キーボードレイアウトの確認

```bash
localectl
# KEYMAP="jp106" であることを確認
```

異なる場合は:
```bash
sudo localectl set-x11-keymap jp106
```

## 4. ネットワーク設定

### 4-1. Wi-Fi接続

- 画面右上のネットワークアイコンをクリック
- Wi-Fiネットワークを選択し、パスワードを入力

### 4-2. IPアドレスの確認

```bash
ip addr show
# または
hostname -I
```

### 4-3. 固定IPの設定（必要な場合）

```bash
# Netplan設定ファイルを作成
sudo nano /etc/netplan/01-netcfg.yaml
```

内容:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

適用:
```bash
sudo netplan apply
```

## 5. システムアップデート

```bash
sudo apt update && sudo apt upgrade -y
```

完了したら再起動:
```bash
sudo reboot
```

## 6. 再起動後の確認

```bash
# OSバージョン
lsb_release -a
# → Ubuntu 24.04.x LTS

# カーネルバージョン
uname -r

# ディスク容量
df -h

# メモリ容量
free -h
```
