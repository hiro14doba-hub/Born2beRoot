*This project has been created as part of the 42 curriculum by hdobashi.*

# Born2beRoot

## Description
本プロジェクトは、Debian GNU/Linuxを用いたセキュアなサーバー環境の構築およびLinuxシステム管理の基礎をまとめたドキュメントです。
物理ディスクの暗号化（LUKS）と論理ボリューム管理（LVM）、PAMを用いた厳格な認証・パスワードポリシー、監査ログ記録を伴うsudo実行環境、UFWによるパケットフィルタリング、非標準ポート（4242）でのSSHリモートアクセス制御、および定期実行監視スクリプト（monitoring.sh）の設計と運用手順について記載しています。

---

## Instructions

### 1. 仮想マシンのセットアップと起動
1. VirtualBoxを起動し、`.vdi` 仮想ディスクイメージを割り当てた仮想マシンを準備します。
2. 仮想マシンのネットワーク設定でアダプター1を「NAT」に設定します。
3. ポートフォワーディング規則に以下を追加します。
   - プロトコル: `TCP`
   - ホストIP: `127.0.0.1` | ホストポート: `4242`
   - ゲストIP: 空欄（または `10.0.2.15`） | ゲストポート: `4242`
4. 仮想マシンを起動し、ディスク暗号化解除プロンプトでLUKSパスフレーズ（`sda5_crypt`）を入力します。
5. ログインプロンプトで一般ユーザー `hdobashi` としてログインします。

### 2. SSHによるリモート接続
ホスト端末（Mac等のターミナル）からポート4242を指定してSSH接続を行います。

```bash
ssh hdobashi@127.0.0.1 -p 4242
```

※セキュリティポリシーに基づき、rootユーザーによる直接のSSH接続は拒否されます。

### 3. ディスク署名（signature.txt）の検証
仮想マシンの起動前に、仮想ディスクファイルのSHA-1ハッシュ値が `signature.txt` と一致していることを検証します。

```bash
# ホスト側のターミナルで実行
shasum ~/VirtualBox\ VMs/hdobashi42/hdobashi42.vdi
diff signature.txt <(shasum ~/VirtualBox\ VMs/hdobashi42/hdobashi42.vdi)
```

---

## System Architecture & Operations Guide

### 1. システム概要と基本仕様
- **仮想化の仕組み:**
  ハイパーバイザー（Type-2）を介して物理ホストマシンのCPU・メモリ・ストレージリソースを論理的に抽象化し、ホストOS上で分離されたゲストOSを安全に並行稼働させています。
- **パッケージ管理システム (apt vs aptitude):**
  - `apt`: コマンドライン操作に特化した高速なパッケージ管理ツール。依存関係の自動解決とインストールを簡潔に行う。
  - `aptitude`: ncursesによるTUI（端末画面UI）を備え、複雑な依存関係の競合が発生した際により柔軟に対話的な選択肢を提示する拡張ツール。
- **アクセス制御フレームワーク (AppArmor vs SELinux):**
  - `AppArmor`: プログラムのファイルシステムパスに基づいてプロファイルを適用する強制アクセス制御（MAC）。設定記述がシンプルで運用管理が容易（Debianで標準採用）。
  - `SELinux`: ファイルやプロセスのinodeおよびセキュリティラベル（コンテキスト）に基づいてアクセス権を制御するMAC。極めて厳格かつ詳細な制御が可能だが設定難度が高い（Rocky/RHELで標準採用）。
- **OSおよびセキュリティモジュール状態確認コマンド:**

```bash
head -n 2 /etc/os-release
sudo aa-status
```

### 2. アカウントと認証ポリシーの運用
- **所属グループの確認:**

```bash
whoami
groups hdobashi
```

- **新規ユーザーの追加登録:**

```bash
sudo adduser <ユーザー名>
```

- **新規グループの作成とユーザー割り当て:**

```bash
sudo addgroup <グループ名>
sudo adduser <ユーザー名> <グループ名>
groups <ユーザー名>
```

- **パスワードポリシー設定の確認:**

```bash
# パスワード有効期限（30日）、最短変更間隔（2日）、警告日数（7日）の確認
grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs | grep -v "^#"

# パスワード文字種・複雑さルールの確認
grep -v "^#" /etc/security/pwquality.conf | grep -v "^$"

# 特定アカウントの有効期限状態確認
sudo chage -l hdobashi
```

- **パスワードポリシー運用の特徴:**
  - *利点:* ブルートフォース攻撃や辞書攻撃を抑止し、認証情報漏洩時の影響期間を制限する。
  - *欠点:* 頻繁な更新や極度な複雑化により、パスワードのメモ化や単純なパターンの使い回しといったセキュリティリスクを誘発しやすい。

### 3. ストレージ構成とホスト設定
- **パーティションおよび暗号化ボリューム構成の確認:**

```bash
lsblk
```

- **LVM（Logical Volume Manager）の構造:**
  物理パーティション（PV）を束ねてボリュームグループ（VG）を形成し、その領域から仮想的な論理ボリューム（LV）を割り当てています。システム稼働中に柔軟な容量拡張・縮小やスナップショット作成が可能です。
- **ホスト名の変更手順:**

```bash
# ホスト名の設定
sudo hostnamectl set-hostname <新しいホスト名>
sudo nano /etc/hosts   # 127.0.1.1 に対応するホスト名を更新

# 設定反映のための再起動
sudo reboot

# 元のホスト名（hdobashi42）への復元
sudo hostnamectl set-hostname hdobashi42
sudo nano /etc/hosts
```

### 4. 権限管理 (sudo) の仕様と監査
- **実装されているセキュリティルール:**
  - 認証試行回数制限（3回）: `passwd_tries=3`
  - パスワード誤入力時のカスタム通知: `badpass_message="Wrong password!"`
  - TTYセッションの強制: `requiretty`
  - コマンド実行パスの制限: `secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"`
  - 全入出力ログの記録: `log_input, log_output`（保存先: `iolog_dir="/var/log/sudo"`）
- **sudoセキュリティ設定の確認:**

```bash
sudo grep -E "passwd_tries|badpass_message|log_input|log_output|iolog_dir|requiretty|secure_path" /etc/sudoers
```

- **監査ログの記録確認:**

```bash
# ログ記録ディレクトリの確認
sudo ls -la /var/log/sudo/00/00/

# コマンド実行によるログ更新テスト
sudo whoami
sudo ls -la /var/log/sudo/00/00/
```

### 5. ネットワークパケットフィルタリング (UFW)
- **ファイアウォールの設計思想:**
  Linuxのパケットフィルタリング機能（iptables/nftables）を管理するUFWを用い、不要な通信ポートを原則遮断した上で、必要なサービスポートのみを限定的に開放しています。
- **現在のルール状態確認（ポート4242のみ開放）:**

```bash
sudo ufw status verbose
```

- **ポートの開放およびルール削除手順:**

```bash
# ポート8080の通信を一時的に許可
sudo ufw allow 8080
sudo ufw status

# 許可ルールの削除
sudo ufw delete allow 8080
sudo ufw status
```

### 6. リモートアクセスデーモン (SSH)
- **SSHデーモンのセキュリティ設定:**
  暗号化通信路による安全な遠隔管理を提供し、既定のポート22から4242へと変更した上で、rootアカウントによる直接ログインを遮断しています。
- **サービス状態およびリッスンポートの確認:**

```bash
sudo systemctl status ssh
ss -tlpn | grep 4242
```

- **接続制御の確認:**

```bash
# 一般ユーザーによる接続テスト
ssh <ユーザー名>@127.0.0.1 -p 4242

# rootログイン無効化設定の確認
grep "PermitRootLogin" /etc/ssh/sshd_config

# rootでの接続拒否テスト
ssh root@127.0.0.1 -p 4242
```

### 7. システム監視スクリプト (monitoring.sh) と定期実行 (cron)
- **スクリプトの配置とコード確認:**

```bash
cat /usr/local/bin/monitoring.sh
```

- **メトリクス収集の処理概要:**
  - OSアーキテクチャ・カーネル情報: `uname -a`
  - 物理CPU数: `/proc/cpuinfo` の `physical id` を抽出・集計
  - 仮想CPU数: `nproc`
  - メモリ使用量・利用率: `free -m` をawkでパース
  - ディスク使用量・利用率: `df -h /` をawkでパース
  - CPU使用率: `top` のアイドル値から算出
  - 最終再起動日時: `who -b`
  - LVM稼働有無: `lsblk` 内のlvmデバイス存在判定
  - アクティブTCP接続数: `ss -ta` で `ESTAB` 状態をカウント
  - ログイン中ユーザー数: `users | wc -w`
  - ネットワークIP/MACアドレス: `hostname -I` および `ip link` から取得
  - sudo実行回数: `journalctl _COMM=sudo` からコマンド実行履歴をカウント
  - 全端末へのブロードキャスト: `wall` コマンドでログイン中の全端末へ通知
- **cronによるスケジュール実行設定:**

```bash
# 10分ごとの実行設定を確認（*/10 * * * *）
sudo crontab -l
```

- **実行スケジュールの変更手順:**

```bash
sudo crontab -e
# 毎分実行とする場合は '*/10 * * * *' を '* * * * *' に変更
```

- **スクリプトファイルを変更しない自動実行の停止・再開手順:**

```bash
# cronサービスを停止・自動起動無効化
sudo systemctl stop cron
sudo systemctl disable cron

# 再起動してスクリプトが自動実行されないことを確認
sudo reboot

# cronサービスを再度有効化・起動
sudo systemctl enable cron
sudo systemctl start cron
```

---

## Project description

### OS選定理由
本システムでは、Debian GNU/Linux（最新安定版）を採用しました。長期的なパッケージの安定稼働実績があり、パッケージ管理システム（`apt`/`dpkg`）の信頼性が高く、堅牢なサーバー環境の構築に適しているためです。

### 比較検証
- **Debian vs Rocky Linux:**
  - *Debian*: コミュニティ主導で開発されるディストリビューション。長期的な安定性を重視し、標準でAppArmorを採用。パッケージ管理には `apt`（.deb）を使用。
  - *Rocky Linux*: RHEL（Red Hat Enterprise Linux）のダウンストリーム互換ディストリビューション。企業向けインフラに適しており、標準でSELinuxを採用。パッケージ管理には `dnf`（.rpm）を使用。
- **AppArmor vs SELinux:**
  - *AppArmor*: ファイルシステムのパス名に基づいてアクセス制限を適用するMAC。プロファイルの設定記述が平易で、保守・運用性に優れる。
  - *SELinux*: ファイルやプロセスのinodeおよびセキュリティラベル（コンテキスト）に基づいてアクセスを制御するMAC。ファイル名変更に強く厳格な制御が可能な一方、運用設計の難度が高い。
- **UFW vs firewalld:**
  - *UFW*: iptables/nftablesのシンプルなフロントエンド。直感的なコマンドライン構文で特定ポートの開閉ルールを即座に構成可能。
  - *firewalld*: ゾーン（zone）ベースの動的ファイアウォールデーモン。通信セッションを切断することなくD-Bus経由で動的にルール変更が可能。
- **VirtualBox vs UTM:**
  - *VirtualBox*: x86/x64アーキテクチャを中心に長年利用されているマルチプラットフォーム対応のType-2ハイパーバイザー。詳細なネットワーク設定やスナップショット管理機能が充実している。
  - *UTM*: Apple Silicon（ARM64）のHypervisorフレームワークおよびQEMUを活用したMac向けのハイパーバイザー。macOS上で高い省電力性とネイティブに近い高速な仮想化性能を提供する。

---

## Resources
- Debian GNU/Linux 公式ドキュメント・管理者ハンドブック
- Linux PAM (`pwquality.conf`) 仕様マニュアル
- Sudoers セキュリティディレクティブ・I/Oログ仕様書
- UFW & OpenSSH サーバーセキュリティ標準ガイド
- [仮想マシンとは](https://cloud.sakura.ad.jp/column/virtual-machine/?gad_source=1&gad_campaignid=17929695792&gbraid=0AAAAADrEfxQ4F7FkHVVLuJpuiYIyGIQjF&gclid=Cj0KCQiAnsvXBhCqARIsAHKgGNZB_N7B0oZDA7HxX6R7Gmmy0A6u1ioQxsBelcdnpkg9iX62T0MA6fkaAglbEALw_wcB)
- [UFWについて](https://qiita.com/Termnix-IT/items/7c26348916d9c55a83cc)
- [debianとは](https://www.miraiserver.ne.jp/column/about_debian/)
- [debian/rocky 比較](https://penguin-gym-linux.com/articles/guides/linux-distro-comparison)


### **AI利用履歴:**
  VirtualBoxのNATポートフォワーディング接続の検証、`/etc/security/pwquality.conf` および `/etc/login.defs` のパラメータ構文確認、ならびに `monitoring.sh` におけるコマンド抽出パイプラインの記述・動作確認において生成AIを活用しました。
