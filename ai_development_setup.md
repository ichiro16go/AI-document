# AI駆動開発スターターガイド：環境構築編（プロ版）

このガイドでは、Windows環境でAIエディタやCLIツールを最大限に活用し、 **「安全」「高速」「高精度」** に開発を行うための基盤構築手順を解説します。

> **次のステップ**: 環境構築が完了したら [AI駆動開発実践編](./ai_development_practice.md) へ進んでください。

---

## 対象読者・前提知識

- **対象**: プログラミング経験があり、AI駆動開発を始めたい方
- **前提知識**:
  - 基本的なターミナル操作（cd, ls, mkdir等）
  - Gitの基本概念（commit, push, pull）
  - 何かしらのプログラミング言語の経験

---

## 目次

1. [Windowsのセットアップ（開発基盤）](#1-windowsのセットアップ開発基盤)
2. [GitHubのセットアップ（セキュリティと連携）](#2-githubのセットアップセキュリティと連携)
3. [AWSの設定（堅牢なデプロイ基盤）](#3-awsの設定堅牢なデプロイ基盤)
4. [AIツールの導入](#4-aiツールの導入cursorとclaude-code)
5. [初心者が守るべき「3つの不変条件」](#付録初心者が守るべき3つの不変条件)
6. [トラブルシューティング](#トラブルシューティング)

---

## 1. Windowsのセットアップ（開発基盤）
WSL環境に移行する理由は以下の通りです。
- Windows標準コマンドとAIが生成するコマンドとの差異による混乱を防ぐため
- Dockerとの親和性が高く、バックエンド開発において優位に働くため
- Pythonのデータサイエンス系のライブラリとの親和性が高いため

### WSL2 (Ubuntu) の導入と「鉄則」

1. Windows Terminalを管理者権限で開き、以下を実行。
   ```powershell
   wsl --install
   ```
   インストール完了後、PCを再起動してください。

2. 再起動後、Ubuntuが自動起動します。ユーザー名とパスワードを設定してください。

3. **Ubuntuの初期設定**（Ubuntu内で実行）
   ```bash
   # パッケージリストの更新とアップグレード
   sudo apt update && sudo apt upgrade -y

   # 開発に必要な基本ツールのインストール
   sudo apt install -y build-essential curl wget git unzip
   ```

4. **【重要：パフォーマンスの鉄則】**
   プロジェクトファイルは必ず**Linux側のホームディレクトリ（`~/` や `/home/ユーザー名/`）**に作成してください。Windows側のフォルダ（`/mnt/c/...`）で作業すると、AIの解析やライブラリのインストールが極端に遅くなります。

   ```bash
   # 開発用ディレクトリの作成（推奨）
   mkdir -p ~/dev
   cd ~/dev
   ```

### ツール構成
- **Windows Terminal**: 視認性を高め、エラーログの読み落としを防ぎます。
- **VS Code + Remote WSL拡張**: Windowsのエディタから、WSL2内の高速なファイルシステムを直接操作します。

### VS Codeの設定

1. [VS Code](https://code.visualstudio.com/)をWindowsにインストール
2. 以下の拡張機能をインストール:
   - **WSL** (Microsoft): WSL内のファイルを直接編集
   - **GitLens**: Git履歴の可視化

3. WSLからVS Codeを開く方法:
   ```bash
   # プロジェクトディレクトリで実行
   code .
   ```

---

## 2. GitHubのセットアップ（セキュリティと連携）

### GitHub CLI (gh command) のインストールと認証

ブラウザ認証だけで連携できるため、SSHキーの設定ミスによる挫折を防ぎます。

```bash
# GitHub CLIのインストール
(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install gh -y

# 認証（ブラウザが開きます）
gh auth login
```

認証時の選択肢:
- `GitHub.com` を選択
- `HTTPS` を選択
- `Login with a web browser` を選択

### 【重要】秘匿情報の管理（.gitignore）
AIにコードを書かせると、誤ってAPIキーなどをソースコードに含めてしまうことがあります。
- **`.env` ファイルの活用**: キーはコードに直接書かず、必ず `.env` に記述します。
- **`.gitignore` の徹底**: プロジェクト開始時に必ず `.gitignore` を作成し、`.env` や `node_modules` をGitの管理から除外してください。

---

## 3. AWSの設定（堅牢なデプロイ基盤）

### IAMユーザーの作成と制限
**ルートユーザーの使用は厳禁です。**
1. 開発用IAMユーザーを作成し、必要な権限（最初は `AdministratorAccess` でも可だが、慣れたら絞る）を付与。
2. **【重要：MFAの設定】**: AWSコンソールから、必ず仮想MFA（スマホアプリ等）を設定してください。アカウント乗っ取りは致命的です。

### AWS CLIの設定と保護
```bash
aws configure
```
- **アクセスキーの取り扱い**: 発行したキーは、絶対にGitHubにプッシュしないでください。AIが「間違えてコミットに含める」リスクを常に意識し、前述の `.gitignore` で防御します。

### AWS Amplify（推奨ルート）
インフラの詳細を意識せず、GitHubとの連携だけで「フロントエンド＋バックエンド」を即座に公開できるため、AI駆動開発のスピード感を最大限に活かせます。

---

## 4. AIツールの導入（CursorとClaude Code）

AI駆動開発には主に2つのツールを使い分けます。

| ツール | 特徴 | 適した用途 |
|--------|------|------------|
| **Cursor** | VS CodeベースのAIエディタ | GUI操作、コード補完、小〜中規模の編集 |
| **Claude Code** | ターミナルベースのAIエージェント | 大規模なリファクタリング、複数ファイルの一括変更、自動化タスク |

### 4.1 Cursorの導入

1. [Cursor公式サイト](https://cursor.sh/)からWindows版をダウンロード・インストール
2. WSL2との連携は自動的に設定されます

#### Cursorのプロジェクト設定（.cursorrules）

プロジェクトのルートに `.cursorrules` を作成し、プロジェクトの全体像を記述します。

```bash
touch .cursorrules
```

```markdown
# プロジェクト概要
このプロジェクトは〇〇です。

# 技術スタック
- フロントエンド: React + TypeScript
- バックエンド: Node.js + Express

# コーディング規約
- 関数はアロー関数で記述
- any型は使用禁止
```

### 4.2 Claude Codeの導入

Claude CodeはAnthropicが提供するターミナルベースのAIコーディングエージェントです。

#### 前提条件: Node.jsのインストール

```bash
# fnm（高速なNode.jsバージョン管理）のインストール
curl -fsSL https://fnm.vercel.app/install | bash

# シェルを再起動するか、以下を実行
source ~/.bashrc

# Node.js LTSのインストール
fnm install --lts
fnm use lts-latest

# 確認
node --version  # v22.x.x などが表示される
npm --version
```

#### Claude Codeのインストール

```bash
# Claude Codeをグローバルにインストール
npm install -g @anthropic-ai/claude-code

# 初回起動（認証が求められます）
claude
```

初回起動時にAnthropicアカウントでの認証が必要です。ブラウザが開くので、指示に従ってください。

#### Claude Codeのプロジェクト設定（CLAUDE.md）

プロジェクトのルートに `CLAUDE.md` を作成すると、Claude Codeが自動的に読み込みます。詳細は[実践編](./ai_development_practice.md)を参照してください。

### 4.3 AIの精度を高めるコツ

1. **コンテキストファイルの作成**: `.cursorrules`（Cursor用）や `CLAUDE.md`（Claude Code用）にプロジェクトの全体像、使用技術、コーディング規約を記述します。
2. **README.mdから書かせる**: 「何を作りたいか」を最初にAIに言語化させることで、AIとの認識のズレを最小限に抑えます。
3. **エラーログの「丸投げ」**: 自分の解釈を挟まず、ターミナルの出力をそのままAIに渡すのが最速の解決法です。

---

## 【付録】初心者が守るべき「3つの不変条件」

1.  **コードに秘密を書かない**: APIキーやパスワードを1回でもGitHubに上げたら、そのキーは「漏洩したもの」として無効化してください。
2.  **Linux環境で完結させる**: ファイル操作、ビルド、テストはすべてWSL2のターミナル内で行ってください。
3.  **AIを盲信しない**: AIの提案が「なぜ動くのか」を、最低限コメントや公式ドキュメントで確認する癖をつけてください。

---

## トラブルシューティング

### WSL2関連

**Q: WSL2のインストールが失敗する**
```powershell
# 仮想化機能が有効か確認（管理者権限のPowerShellで実行）
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
# その後、PCを再起動
```

**Q: WSL2が極端に遅い**
- Windows側のフォルダ（`/mnt/c/...`）で作業していないか確認
- Windows Defenderの除外設定にWSLのパスを追加

**Q: `code .` でVS Codeが開かない**
```bash
# VS CodeのPATHを再設定
export PATH="$PATH:/mnt/c/Users/$(whoami)/AppData/Local/Programs/Microsoft VS Code/bin"
```

### GitHub CLI関連

**Q: `gh auth login` でブラウザが開かない**
```bash
# デバイスコードフローを使用
gh auth login --web
```

### Claude Code関連

**Q: `claude` コマンドが見つからない**
```bash
# npmのグローバルパスを確認
npm config get prefix
# PATHに追加されているか確認
echo $PATH
# 必要に応じて.bashrcに追加
echo 'export PATH="$PATH:$(npm config get prefix)/bin"' >> ~/.bashrc
source ~/.bashrc
```

**Q: Claude Codeの認証が失敗する**
```bash
# 認証情報をリセット
claude logout
claude login
```

**Q: AIが古い情報を提案してくる**
- ライブラリのバージョンを明示的に伝える
- 「2024年以降の方法で」などと指定する
- 公式ドキュメントのURLを一緒に渡す

---

## 次のステップ

環境構築が完了したら、[AI駆動開発実践編](./ai_development_practice.md)に進んで、実際の開発フローを学びましょう。
