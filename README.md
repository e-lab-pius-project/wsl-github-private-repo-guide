# WSL Ubuntu で GitHub にログインし、プライベートリポジトリを取得する

Windows 上の WSL Ubuntu から GitHub にログインし、アクセス権のあるプライベートリポジトリを取得する手順です。GitHub CLI（`gh`）でブラウザ認証を行い、Git の通信には HTTPS を使います。

初回は 1〜4 を順番に実施します。2 回目以降の更新は「5. 取得済みのリポジトリを更新する」、エラーが出たときは「6. 困ったとき」を参照してください。

ここでいう「プロジェクト」はソースコードや文書を保管する GitHub のリポジトリを指します。初回の取得は `git clone`、取得済みのフォルダーを更新する操作は `git pull` です。

## 始める前に用意するもの

- WSL と Ubuntu を導入した Windows PC。本書は WSL 2 と Ubuntu 24.04 LTS 以降を想定します。
- 自分の GitHub アカウントと、対象リポジトリの読み取り権限。
- 対象の HTTPS URL。例：`https://github.com/OWNER/REPOSITORY.git`。
- インターネット接続と、Ubuntu で `sudo` を実行できるユーザー。

招待メールを受け取っている場合は、先に招待を承諾してください。Windows のブラウザで対象リポジトリを開き、内容が見えることを確かめます。ログインするだけではプライベートリポジトリの権限は付与されません。

本書の `OWNER` は所有者のユーザー名または Organization 名、`REPOSITORY` はリポジトリ名です。自分が取得する対象の値へ置き換えてください。

## 1. Windows から Ubuntu を起動する

Windows の **PowerShell** で、導入済みのディストリビューション名を確認します。

```powershell
wsl --list --verbose
```

一覧の名前に合わせて起動します。次は `Ubuntu-24.04` の例です。名前が `Ubuntu` や `Ubuntu-26.04` の場合は、その名前に置き換えます。

```powershell
wsl -d Ubuntu-24.04 --cd ~
```

Ubuntu が未導入の場合は、[Microsoft の WSL インストール手順](https://learn.microsoft.com/windows/wsl/install)を先に実施してください。

**ここから先の `bash` コードブロックは、Ubuntu のターミナルで実行します。** PowerShell や Git Bash には貼り付けません。

```bash
cat /etc/os-release
whoami
pwd
```

Ubuntu であることと、自分の通常ユーザーで起動していることを確認します。`whoami` が `root` なら、通常ユーザーで入り直してください。GitHub の認証や `clone` に `sudo` は付けません。

## 2. Ubuntu に Git と GitHub CLI を導入する

Git はファイルの取得と更新、GitHub CLI は GitHub へのログインに使います。まず Git と、パッケージ取得に必要なツールを導入します。

```bash
sudo apt update && sudo apt install -y git curl ca-certificates
```

`sudo` のパスワードは **Ubuntu のユーザーパスワード**です。入力中は文字が表示されません。

次に GitHub CLI の公式 APT リポジトリを登録します。以下は [GitHub CLI の Linux 導入手順](https://github.com/cli/cli/blob/trunk/docs/install_linux.md)に沿ったコマンドです。ブロック全体を貼り付け、エラーになった場合は先へ進まず原因を確認してください。

```bash
sudo install -d -m 755 /etc/apt/keyrings /etc/apt/sources.list.d &&
keyring_file=$(mktemp) &&
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg -o "$keyring_file" &&
sudo install -m 644 "$keyring_file" /etc/apt/keyrings/githubcli-archive-keyring.gpg &&
rm -- "$keyring_file" &&
printf 'deb [arch=%s signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main\n' "$(dpkg --print-architecture)" |
  sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null &&
sudo apt update &&
sudo apt install -y gh
```

導入結果と実行ファイルの場所を確認します。

```bash
git --version
gh --version
command -v git
command -v gh
```

上の手順で導入した場合、場所はそれぞれ `/usr/bin/git`、`/usr/bin/gh` です。Windows 側に Git や GitHub CLI が導入済みでも、本書では Ubuntu 側のものを使います。

## 3. ブラウザで GitHub にログインする

Ubuntu で次を実行します。

```bash
gh auth login --hostname github.com --git-protocol https --web
```

1. Git の認証にも GitHub の資格情報を使うか聞かれたら、`Y` を選びます。
2. ターミナルに表示される一時コードを確認し、案内に従ってブラウザを開きます。
3. ブラウザが自動で開かない場合は、Windows のブラウザで [GitHub のデバイス認証画面](https://github.com/login/device)を開き、表示されたコードを入力します。
4. **対象リポジトリにアクセスできる自分のアカウント**であることを確認し、GitHub CLI のアクセスを承認します。2 要素認証が求められたら、その手続きも行います。
5. Ubuntu のターミナルに戻り、コマンドの完了を待ちます。

画面の文言はバージョンなどで異なる場合があります。ブラウザを閉じるだけでは認証完了とは限らないため、次の確認まで進めてください。

```bash
gh auth status --hostname github.com
gh auth setup-git --hostname github.com
```

`gh auth status` で、意図したアカウントが有効になっていることを確認します。`gh auth setup-git` は、Git が GitHub CLI の認証情報を使うように設定します。コマンドの役割は [ログインの公式説明](https://cli.github.com/manual/gh_auth_login)と [Git 連携の公式説明](https://cli.github.com/manual/gh_auth_setup-git)で確認できます。

### 認証情報の保存先を確認する

GitHub CLI は利用可能な資格情報ストアに認証情報を保存します。ストアを利用できない環境では、設定ファイルへの平文保存に切り替わります。WSL でも実際の保存先を `gh auth status` で確認してください。[保存方法の公式説明](https://docs.github.com/en/github-cli/github-cli/authenticating-with-github-cli)

平文保存の場合、そのファイルを読める人は認証情報も読めます。共有の Ubuntu ユーザーでログインしたり、`~/.config/gh/` などの認証設定をリポジトリや共有フォルダーへコピーしたりしないでください。認証コードやトークンをチャット、スクリーンショット、手順書に載せることも避けます。

## 4. プライベートリポジトリを初めて取得する

Ubuntu のホームディレクトリ内に作業場所を用意します。Linux のツールで作業するファイルは、WSL の Linux ファイルシステム内に置くと性能面で有利です。[Microsoft の保存場所の説明](https://learn.microsoft.com/windows/wsl/filesystems)

```bash
mkdir -p ~/projects
cd ~/projects
```

以下の URL の `OWNER` と `REPOSITORY` を置き換えて実行します。URL は GitHub のリポジトリ画面で **Code → HTTPS** からコピーできます。

```bash
git clone https://github.com/OWNER/REPOSITORY.git
```

取得に成功すると `REPOSITORY` というフォルダーが作成されます。その中へ移動し、接続先と作業状態を確認します。

```bash
cd ~/projects/REPOSITORY
git remote -v
git status
git branch -vv
```

`origin` が目的の HTTPS URL を指し、`git status` で作業ツリーに変更がないことを確認します。通常の `clone` では既定ブランチと、その追跡先が設定されます。ブランチ名はリポジトリによって異なるため、`main` と決めつけません。

Windows のエクスプローラーでこのフォルダーを開く場合は、Ubuntu で次を実行します。

```bash
explorer.exe .
```

## 5. 取得済みのリポジトリを更新する

2 回目以降は、Ubuntu で取得済みのフォルダーに入り、現在のブランチと変更内容を確認します。ログインが有効なら、毎回ログインし直す必要はありません。

```bash
cd ~/projects/REPOSITORY
git status
git branch -vv
```

作業ツリーに変更がなく、更新したいブランチにいることを確認してから実行します。

```bash
git pull --ff-only
git status
```

`--ff-only` は、履歴が分岐している場合に自動マージせず停止する指定です。`Already up to date.` は取り込む更新がないことを表します。取得するのは現在のブランチに設定された追跡先の変更です。[Git の pull の説明](https://git-scm.com/docs/git-pull)

ローカルに編集がある場合は、先に内容を確認してコミットするか、必要なファイルを別の場所へ退避してください。`pull` が止まっても、変更を消すコマンドやフォルダー削除で解決しようとせず、次の表を参照します。

## 6. 困ったとき

| 症状 | 確認・対処 |
| --- | --- |
| `gh: command not found` | Ubuntu で手順 2 を実施したか確認します。PowerShell と Ubuntu の導入先は別です。 |
| ブラウザが開かない | 手順 3 のデバイス認証画面を Windows のブラウザで開き、ターミナルのコードを入力します。期限切れならログインコマンドからやり直します。 |
| `Repository not found` / 404 | URL、ログイン中のアカウント、招待の承諾、読み取り権限を確認します。ブラウザでも同じアカウントで対象を開きます。 |
| 403、Organization の承認を求められる | Organization の SSO や OAuth アプリ制限に従います。アクセスできなければ Organization 管理者へ GitHub CLI の利用可否を確認します。 |
| HTTPS なのにユーザー名・パスワードを求められる | `gh auth status --hostname github.com` と `gh auth setup-git --hostname github.com` を再確認します。GitHub のアカウントパスワードを入力しても Git の HTTPS 認証には使えません。 |
| `Permission denied (publickey)` | SSH の URL を使っている可能性があります。`git remote -v` を確認し、本書の HTTPS URL に合わせます。 |
| `destination path ... already exists` | 同名フォルダーがあります。既存の Git リポジトリなら手順 5 へ進みます。別の用途のフォルダーなら、内容を確認し、違う取得先名を指定します。削除は不要です。 |
| `not a git repository` | `pwd` を確認し、`cd ~/projects/REPOSITORY` で取得先へ移動します。ZIP を展開したフォルダーには通常 Git の管理情報がありません。 |
| `Your local changes ... would be overwritten` | `git status` と `git diff` で編集を確認し、コミットまたは退避してから再実行します。未追跡ファイルも `git status` で確認します。 |
| `Not possible to fast-forward` | ローカルと接続先の履歴が分岐しています。`git status` と `git log --oneline --graph --decorate -10` を確認し、担当者と統合方法を相談します。 |
| `There is no tracking information` | `git branch -vv` と `git branch -r` で対象を確認します。追跡先が不明なら、推測で設定せずリポジトリ管理者へ確認します。 |

クローン時の認証・権限エラーについては、[GitHub のトラブルシューティング](https://docs.github.com/en/repositories/creating-and-managing-repositories/troubleshooting-cloning-errors)も参照してください。

### SSH の接続先を HTTPS に変更する場合

取得済みフォルダーで、`OWNER` と `REPOSITORY` を実際の値に置き換えます。

```bash
git remote set-url origin https://github.com/OWNER/REPOSITORY.git
git remote -v
```

### 間違ったアカウントでログインした場合

ログイン済みのアカウント一覧を確認します。

```bash
gh auth status --hostname github.com
```

目的のアカウントが登録済みなら、`YOUR_GITHUB_LOGIN` をそのログイン名に置き換えて切り替えます。

```bash
gh auth switch --hostname github.com --user YOUR_GITHUB_LOGIN
```

未登録なら、手順 3 を再実施し、ブラウザで目的のアカウントを選びます。ブラウザと CLI は別のアカウントになり得るため、認証後に再度 `gh auth status` を確認します。

### 解決しないときに伝える情報

リポジトリ管理者または授業・プロジェクトの担当者へ、実行したコマンド、エラー文、`git --version` と `gh --version` を伝えてください。認証トークンや一時コードは伝えません。非公開のリポジトリ名や URL を含む出力は、共有してよい相手か確認します。

## 作業を終えたら

自分専用の PC・Ubuntu ユーザーなら、次回に備えてログイン状態を維持できます。ローカルのログイン情報を削除する場合は、次を実行します。

```bash
gh auth logout --hostname github.com
```

ログアウトしても、取得したファイルは PC に残ります。この操作は GitHub 側のトークン失効とは別です。認証情報を漏らした場合は、GitHub の設定で該当する認証情報やアプリの認可を取り消してください。[ログアウトの公式説明](https://cli.github.com/manual/gh_auth_logout)

## 用語

| 用語 | 意味 |
| --- | --- |
| WSL | Windows 上で Linux 環境を動かす仕組み。Ubuntu はその環境として利用する Linux ディストリビューションです。 |
| リポジトリ | ファイルと変更履歴を保管する単位。プライベートの場合はアクセス権が必要です。 |
| Git / GitHub CLI | Git は変更履歴の管理と取得・更新を行うツール。GitHub CLI（`gh`）は GitHub の認証や各種操作を行うツールです。 |
| clone / pull | `clone` はローカルへの初回取得、`pull` は現在のブランチへの更新取り込みです。 |
| origin / 追跡先 | `origin` は通常、取得元につく接続先名。追跡先はローカルブランチが更新元として参照するリモートのブランチです。 |

## 出典と確認範囲

本文中のリンクは GitHub、Git、Microsoft の公式資料です。確認日：2026 年 11 月 6 日。

本書は HTTPS とブラウザ認証による取得・更新を対象とします。SSH 鍵の設定、手動での個人アクセストークン発行、変更の push、プロジェクト固有のビルド手順は扱いません。

作成時には公式資料との照合と Bash コマンドの構文確認を実施しました。新規 Ubuntu へのインストール、対話式の GitHub ログイン、実際のプライベートリポジトリの clone・pull は通しで実施していません。Organization ごとの承認条件は管理者へ確認してください。
