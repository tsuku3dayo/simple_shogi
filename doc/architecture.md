# システム構成と技術スタック

戦国将棋（`simple-shogi`）の仕組みと、使っている技術・ツール・プラットフォームのまとめ。

- 調べた時点のバージョン: v1.3.2（`package.json`）
- 開発のルールは [`CLAUDE.md`](../CLAUDE.md) を参照

## 全体像

**サーバーを持たない、ブラウザだけで動く PWA**（静的サイト）。ルール・CPU・物語・音・保存は、すべて端末の中で処理し、外部への通信はない。

```
GitHub (main) ──push──▶ GitHub Actions ──テスト通過──▶ GitHub Pages（静的配信）
                                                          │
                                         ブラウザ / ホーム画面アプリ
                                         ├─ index.html + public/boot.js（起動の見張り）
                                         ├─ Vite がまとめた TypeScript（src/*）
                                         ├─ Service Worker（Workbox、オフライン用キャッシュ）
                                         └─ localStorage（記録・途中の対局）
```

## 技術スタック

| 分野 | 使っているもの |
|---|---|
| 言語 | TypeScript ~6.0（UI フレームワークなし。DOM を直接操作） |
| ビルド | Vite 8（`base: './'` で GitHub Pages のサブパスに対応） |
| PWA | vite-plugin-pwa 2 + workbox-window（`registerType: 'prompt'`: 新しい版は「更新」ボタンで切り替え） |
| 単体テスト | Vitest 5（Node 環境、`tests/*.test.ts`） |
| 画面テスト | Playwright（Pixel 7 のエミュレーション、build 済みの成果物を `vite preview` で配信してテスト） |
| Lint | ESLint 10 + typescript-eslint（`innerHTML` への代入を禁止。フォーマッタは入れていない） |
| 実行環境 | Node.js 24（CI と同じ版） |
| 音 | Web Audio API で合成（効果音 `sound.ts`、BGM `music.ts` / `music-data.ts`）。音声ファイルなし |
| 絵 | 武将の顔は SVG をコードで生成（`portrait.ts`）。アイコンは `scripts/make-icons.mjs` で生成 |

## コードの層

### ロジック層（DOM に触れない純関数。`tests/` で単体テスト）

| ファイル | 役割 |
|---|---|
| `src/shogi.ts` | 盤（`Uint8Array` の 81 マス）、合法手の生成、王手の判定。perft（合法手の数え上げ）テストで正しさを確認 |
| `src/ai.ts` | CPU の思考。α-β 探索 + 反復深化（持ち時間の中で読む深さを 1 手ずつ増やす）。強さは 8 段階（深さ・時間・「うっかり」の大きさ）。Web Worker は使わず、画面と同じスレッドで動く |
| `src/game.ts` | 棋譜、終局の判定、保存用の符号化 |
| `src/story.ts` / `src/story-data/` | 物語「戦国三英雄伝」（信長・秀吉・家康の 3 編、全 24 話）のデータ |
| `src/talk.ts` | 対局中のセリフ選び |
| `src/tutorial.ts` | チュートリアルのレッスンと目標の判定 |
| `src/backup.ts` | バックアップの形式と版番号（`SCHEMA_VERSION`） |

### 画面層（DOM の操作だけ）

| ファイル | 役割 |
|---|---|
| `src/main.ts` | 画面の切り替え（タイトル・物語・自由対局・対局・会話・レッスン）と起動 |
| `src/play.ts` | 対局画面 |
| `src/tutorial-view.ts` | チュートリアルの練習盤 |
| `src/scene.ts` / `src/portrait.ts` | 会話シーンと武将の顔 |
| `src/pieces.ts` / `src/ui.ts` | 駒の見た目、共通の小物 |
| `src/style.css` | 見た目（色は `:root` の変数に集約。暗い固定の配色） |

### 基盤

| ファイル | 役割 |
|---|---|
| `src/storage.ts` | `localStorage` を `try/catch` で包んだ保存。キーは `simple-shogi-` で始まる。`migrate()` で古い形の保存データを移行。書き出し / 読み込み（バックアップ） |
| `src/pwa.ts` | Service Worker の登録と「更新」ボタン |
| `src/install.ts` | インストール方法の説明（iPhone / Android / パソコン） |
| `public/boot.js` | Vite を通らない普通のスクリプト。起動に失敗したら、このアプリの Service Worker とキャッシュだけを消して読み込み直す（保存データは消さない） |

## ツールとプラットフォーム

| 用途 | 使っているもの |
|---|---|
| ソースコードの管理 | GitHub（作業は feature ブランチ → PR → main） |
| 公開 | GitHub Pages（https://mmishina-art.github.io/simple_shogi/） |
| CI | GitHub Actions `test.yml`: main への push と全 PR で lint / 単体テスト / build / check:dist / E2E。失敗時は Playwright のレポートを保存 |
| 公開の自動化 | GitHub Actions `pages.yml`: main への push で同じ検査をし、通ったときだけ `dist/` を公開 |
| 依存の更新 | Dependabot（`.github/dependabot.yml`。`vite`・`vitest`・`vite-plugin-pwa` は 1 つの PR にまとめる） |
| PWA の成立チェック | `scripts/check-dist.mjs`（manifest、アイコン、バージョン表示、インストール方法の入口、`boot.js`、ズーム禁止がないことを確認） |
| 開発支援 | Claude Code（ルールは `CLAUDE.md`） |

## 設計の特徴

- **フレームワークもバックエンドも入れない。** 依存を減らし、オフラインでも確実に動かすため。
- **同じサイトに複数のアプリが載る前提。** アプリの ID（`package.json` の `name`）を、保存キー・バックアップの `app` 名・キャッシュを消す範囲の 3 か所に使い、ほかのアプリと混ざらないようにしている。
- **バージョンの元は 1 か所。** `package.json` の `version` を `__APP_VERSION__` としてタイトル画面に表示する。
- **更新はプレイ中に勝手に切り替えない。** 新しい版は待機させ、「更新」ボタンで切り替える。

## 起動に必要なもの

### 遊ぶだけ

インストールは不要。ブラウザで公開 URL を開く。ホーム画面に追加すると、アプリのように開けてオフラインでも遊べる（任意）。

### 手元で開発する

| 必要なもの | 用途 | 入れ方 |
|---|---|---|
| Node.js 24（npm も入る） | 開発サーバー・ビルド・テスト | https://nodejs.org または `winget install OpenJS.NodeJS --version 24` |
| 依存パッケージ | Vite・TypeScript など | `npm ci` |
| Chromium（Playwright 用） | 画面テストのときだけ | `npx playwright install chromium`（初回のみ） |
| Git | コミット・PR | Sourcetree に同梱の Git を使える（下を参照）。別に入れるなら `winget install Git.Git` |

#### Sourcetree に同梱の Git をターミナルで使う

Sourcetree は自分用の Git を `%LOCALAPPDATA%\Atlassian\SourceTree\git_local\` に持っている。PATH に入っていないので、そのままではターミナルから `git` と打っても見つからない。PowerShell で次を 1 回実行し、ターミナルを開き直すと使えるようになる。

```
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$env:LOCALAPPDATA\Atlassian\SourceTree\git_local\cmd", "User")
```

`git --version` でバージョンが出れば完了。

#### 起動

```
npm ci
npm run dev
```

表示された URL（ふつうは `http://localhost:5173/`）をブラウザで開く。

| やりたいこと | コマンド |
|---|---|
| 開発サーバー | `npm run dev` |
| 型チェック + ビルド | `npm run build` |
| Lint | `npm run lint` |
| 単体テスト | `npm test` |
| 画面テスト | `npm run test:e2e` |
| まとめて確認 | `npm run verify` |
