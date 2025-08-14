# Windows汎用的な開発環境構築ガイド

このドキュメントでは、クリーンなWindows環境から汎用的な開発環境を構築する手順を、開発初心者向けに詳細に解説します。

## 目次

1. [概要](#概要)
2. [Windows Subsystem for Linux 2 (WSL2) の導入](#wsl2の導入)
3. [Visual Studio Code (VSCode) の導入と設定](#vscodeの導入と設定)
4. [Node.js 開発環境の構築 (nvm経由)](#nodejs開発環境の構築)
5. [Python3 開発環境の構築 (pyenv経由)](#python3開発環境の構築)
6. [推奨ソフトウェアとツール](#推奨ソフトウェアとツール)
7. [VSCode 推奨拡張機能](#vscode推奨拡張機能)
8. [トラブルシューティング](#トラブルシューティング)

## 概要

このガイドでは以下の環境を構築します：

- **WSL2**: Linux環境をWindows上で動作させる
- **Ubuntu**: WSL2上のLinuxディストリビューション
- **VSCode**: メインのエディタ・統合開発環境
- **Node.js**: JavaScriptランタイム（nvmでバージョン管理）
- **Python3**: プログラミング言語（pyenvでバージョン管理）

この構成により、モダンな開発環境でバージョン切り替えが容易な環境が整います。

## WSL2の導入

### 前提条件

- Windows 10 version 2004以降、またはWindows 11
- 管理者権限でのコマンド実行が可能

### Step 1: WSLとVirtualMachinePlatform機能の有効化

1. **PowerShellを管理者として実行**
   - スタートメニューで「PowerShell」を検索
   - 「Windows PowerShell」を右クリック
   - 「管理者として実行」を選択

2. **WSL機能を有効化**
   ```powershell
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   ```

3. **Virtual Machine Platform機能を有効化**
   ```powershell
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   ```

4. **コンピューターを再起動**
   ```powershell
   Restart-Computer
   ```

### Step 2: WSL2をデフォルトバージョンに設定

1. **再起動後、PowerShellを管理者として再実行**

2. **WSL2をデフォルトに設定**
   ```powershell
   wsl --set-default-version 2
   ```

### Step 3: Linuxカーネル更新プログラムのインストール

1. **WSL2 Linuxカーネル更新プログラムをダウンロード**
   - [Microsoft公式サイト](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)からダウンロード
   
2. **ダウンロードしたmsiファイルを実行してインストール**

### Step 4: Ubuntuディストリビューションのインストール

1. **Microsoft Storeを開く**
   - スタートメニューから「Microsoft Store」を検索して開く

2. **Ubuntuを検索してインストール**
   - 検索バーに「Ubuntu」と入力
   - 「Ubuntu」（最新LTS版）を選択
   - 「入手」をクリックしてインストール

3. **Ubuntuの初期設定**
   - インストール完了後、「起動」をクリック
   - 初回起動時にユーザー名とパスワードを設定
   ```
   例：
   ユーザー名: 自分の名前（なんでもいい）
   パスワード: 任意の安全なパスワード（ローカル環境だからユーザー名と同じでも可）
   ```

### Step 5: WSL2の動作確認

1. **PowerShellでWSLのバージョン確認**
   ```powershell
   wsl --list --verbose
   ```
   
   正常にインストールされていれば以下のような出力が表示されます：
   ```
   NAME      STATE           VERSION
   * Ubuntu    Running         2
   ```

## VSCodeの導入と設定

### Step 1: Visual Studio Codeのインストール

1. **VSCode公式サイトからダウンロード**
   - [https://code.visualstudio.com/](https://code.visualstudio.com/)にアクセス
   - 「Download for Windows」をクリック

2. **インストーラーを実行**
   - ダウンロードしたファイルを実行
   - インストール時の重要な設定：
     - ✅ 「Add to PATH」にチェック
     - ✅ 「Register Code as an editor for supported file types」にチェック
     - ✅ 「Add "Open with Code" action to Windows Explorer file context menu」にチェック
     - ✅ 「Add "Open with Code" action to Windows Explorer directory context menu」にチェック

### Step 2: WSL拡張機能のインストール

1. **VSCodeを起動**

2. **WSL拡張機能をインストール**
   - サイドバーの拡張機能アイコン（四角が4つのアイコン）をクリック
   - 検索バーに「WSL」と入力
   - 「WSL」拡張機能（Microsoft製）をインストール

### Step 3: WSL環境でVSCodeを使用する設定

1. **WSL環境でVSCodeを起動**
   - Ubuntuターミナルを開く
   - 以下のコマンドでVSCodeをWSL環境で起動
   ```bash
   code .
   ```

2. **初回起動時の設定**
   - VSCode Serverが自動的にWSL環境にインストールされます
   - 数分待つとWSL環境でVSCodeが使用可能になります

## Node.js開発環境の構築

### Step 1: nvmのインストール

1. **Ubuntuターミナルを開く**

2. **curlのインストール（未インストールの場合）**
   ```bash
   sudo apt update
   sudo apt install curl
   ```

3. **nvmのインストール**
   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
   ```

4. **ターミナルを再起動またはシェル設定を再読み込み**
   ```bash
   source ~/.bashrc
   ```

5. **nvmの動作確認**
   ```bash
   nvm --version
   ```

### Step 2: Node.jsのインストール

1. **利用可能なNode.jsバージョンの確認**
   ```bash
   nvm list-remote
   ```

2. **最新のLTS版Node.jsをインストール**
   ```bash
   nvm install --lts
   nvm use --lts
   ```

3. **デフォルトバージョンの設定**
   ```bash
   nvm alias default node
   ```

4. **インストールの確認**
   ```bash
   node --version
   npm --version
   ```

### Step 3: Node.js開発環境の最適化

1. **npmの最新バージョンへの更新**
   ```bash
   npm install -g npm@latest
   ```

2. **よく使用されるグローバルパッケージのインストール**
   ```bash
   npm install -g yarn
   npm install -g typescript
   npm install -g @angular/cli
   npm install -g create-react-app
   npm install -g vue@next
   ```

## Python3開発環境の構築

### Step 1: pyenvの依存関係インストール

1. **必要なパッケージのインストール**
   ```bash
   sudo apt update
   sudo apt install -y make build-essential libssl-dev zlib1g-dev \
   libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
   libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-openssl git
   ```

### Step 2: pyenvのインストール

1. **pyenvのクローン**
   ```bash
   git clone https://github.com/pyenv/pyenv.git ~/.pyenv
   ```

2. **シェル設定ファイルの編集**
   ```bash
   echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
   echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
   echo 'eval "$(pyenv init -)"' >> ~/.bashrc
   ```

3. **シェル設定の再読み込み**
   ```bash
   source ~/.bashrc
   ```

4. **pyenvの動作確認**
   ```bash
   pyenv --version
   ```

### Step 3: Pythonのインストール

1. **利用可能なPythonバージョンの確認**
   ```bash
   pyenv install --list
   ```

2. **最新の安定版Pythonのインストール**
   ```bash
   pyenv install 3.11.6
   pyenv install 3.12.0
   ```

3. **グローバルバージョンの設定**
   ```bash
   pyenv global 3.12.0
   ```

4. **インストールの確認**
   ```bash
   python --version
   pip --version
   ```

### Step 4: Python開発環境の最適化

1. **pipの最新バージョンへの更新**
   ```bash
   pip install --upgrade pip
   ```

2. **よく使用されるパッケージのインストール**
   ```bash
   pip install virtualenv
   pip install pipenv
   pip install poetry
   pip install jupyter
   pip install pytest
   pip install black
   pip install flake8
   pip install mypy
   ```

## 推奨ソフトウェアとツール

### 必須ツール

1. **Git**
   ```bash
   sudo apt install git
   ```
   
   設定例：
   ```bash
   git config --global user.name "あなたの名前"
   git config --global user.email "your.email@example.com"
   ```

2. **Docker Desktop for Windows**
   - [Docker公式サイト](https://www.docker.com/products/docker-desktop)からダウンロード
   - WSL2バックエンドを有効化

### 開発効率向上ツール

1. **Windows Terminal**
   - Microsoft Storeから「Windows Terminal」をインストール
   - モダンなターミナル環境

2. **PowerToys**
   - [Microsoft PowerToys](https://github.com/microsoft/PowerToys)
   - Windows用生産性ツール

3. **zsh + Oh My Zsh（オプション）**
   ```bash
   sudo apt install zsh
   sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
   ```

### ブラウザとデバッグツール

1. **Google Chrome**
   - 開発者ツールが充実

2. **Firefox Developer Edition**
   - 開発者向け機能

### データベースツール

1. **DBeaver**
   - 多くのデータベースに対応したGUIツール

## VSCode推奨拡張機能

### 必須拡張機能

1. **WSL**
   - Microsoft製 - WSL環境での開発に必須

2. **Remote Development**
   - Microsoft製 - リモート開発環境のサポート

### 言語サポート

1. **JavaScript/TypeScript**
   - JavaScript (ES6) code snippets
   - TypeScript Importer
   - Prettier - Code formatter
   - ESLint

2. **Python**
   - Python (Microsoft製)
   - Python Docstring Generator
   - autoDocstring
   - Python Indent

3. **HTML/CSS**
   - HTML CSS Support
   - Auto Rename Tag
   - Live Server
   - IntelliSense for CSS class names in HTML

### Git関連

1. **GitLens — Git supercharged**
   - Gitの詳細情報表示

2. **Git History**
   - Git履歴の視覚化

### 生産性向上

1. **Bracket Pair Colorizer 2**
   - 括弧の色分け

2. **indent-rainbow**
   - インデントの可視化

3. **Material Icon Theme**
   - ファイルアイコンテーマ

4. **One Dark Pro**
   - ダークテーマ

5. **Auto Save**
   - 自動保存機能

### コード品質

1. **SonarLint**
   - コード品質チェック

2. **Code Spell Checker**
   - スペルチェック

### ファイル管理

1. **Path Intellisense**
   - パス補完

2. **filesize**
   - ファイルサイズ表示

### 拡張機能インストール方法

1. **VSCode内でのインストール**
   - Ctrl + Shift + X で拡張機能パネルを開く
   - 拡張機能名を検索してインストール

2. **コマンドラインでのインストール**
   ```bash
   code --install-extension ms-vscode-remote.remote-wsl
   code --install-extension ms-python.python
   code --install-extension esbenp.prettier-vscode
   ```

## トラブルシューティング

### WSL2関連

**問題**: WSL2が起動しない
**解決策**:
1. Hyper-Vが有効になっているか確認
2. BIOSで仮想化技術が有効になっているか確認
3. Windows機能でWSLとVirtual Machine Platformが有効か確認

**問題**: メモリ使用量が多い
**解決策**:
1. ホームディレクトリに`.wslconfig`ファイルを作成
   ```ini
   [wsl2]
   memory=4GB
   processors=2
   ```

### Node.js関連

**問題**: nvmコマンドが見つからない
**解決策**:
```bash
source ~/.bashrc
# または
source ~/.zshrc
```

**問題**: 権限エラーでnpmパッケージがインストールできない
**解決策**:
```bash
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Python関連

**問題**: pyenvでPythonのインストールが失敗する
**解決策**:
1. 依存関係パッケージの再インストール
2. システムの更新
   ```bash
   sudo apt update && sudo apt upgrade
   ```

### VSCode関連

**問題**: WSL環境でVSCodeが起動しない
**解決策**:
1. WSL拡張機能がインストールされているか確認
2. ファイアウォール設定の確認
3. VSCodeの再インストール

---

このドキュメントに従って環境構築を行うことで、WSL2ベースの効率的な開発環境が構築できます。各ステップで問題が発生した場合は、該当するトラブルシューティングセクションを参照してください。

また、開発プロジェクトに応じて、追加のツールや設定が必要になる場合があります。プロジェクト固有の要件については、各プロジェクトのドキュメントを参照してください。