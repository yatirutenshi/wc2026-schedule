# ワールドカップ2026 全試合カレンダー（日本時間・自動更新）

全104試合のキックオフ（JST）・地上波放送・試合結果を表示する静的サイト。
GitHub Actions が30分ごとに [football-data.org](https://www.football-data.org/) のAPIをチェックし、
試合結果と決勝トーナメントの対戦カードを `matches.json` に自動反映します。

## 仕組み

```
football-data.org API
      │  30分ごとに取得（GitHub Actions）
      ▼
matches.json を更新 → git commit & push
      │
      ▼
Netlify がpushを検知して自動デプロイ
      │
      ▼
index.html が matches.json を読み込んで表示
```

- スコアが入ると「VS」がスコアバッジに変わり、勝者に下線が付きます
- 決勝Tの「F組1位」などのプレースホルダーは、対戦カード確定後に自動で国名へ置き換わります
- キックオフ時刻が変更された場合も自動で追従します
- API取得に失敗してもサイトは最後に成功したデータで表示され続けます（壊れません）

## セットアップ手順

### 1. football-data.org のAPIキーを取得（無料）

1. https://www.football-data.org/client/register でメール登録
2. 届いたメールに記載の **API token** を控える
   （無料プランでFIFAワールドカップに対応。10リクエスト/分まで）

### 2. GitHubリポジトリを作成してこのフォルダをpush

```bash
git init
git add .
git commit -m "initial"
git branch -M main
git remote add origin https://github.com/＜ユーザー名＞/＜リポジトリ名＞.git
git push -u origin main
```

### 3. APIキーをSecretsに登録

リポジトリの **Settings → Secrets and variables → Actions → New repository secret**

- Name: `FOOTBALL_DATA_TOKEN`
- Secret: 手順1のAPIトークン

### 4. Netlifyを既存プロジェクトとGit連携

1. Netlifyの **Project configuration → Build & deploy → Continuous deployment**
2. **Link repository** からGitHubリポジトリを選択
3. Build command: 空欄 / Publish directory: `/`（リポジトリ直下）

→ 既存のURL（◯◯.netlify.app）のまま、push毎に自動デプロイされるようになります。

### 5. 動作確認

リポジトリの **Actions タブ → 「W杯結果の自動更新」→ Run workflow** で手動実行。
ログに「APIから ◯◯ 試合を取得しました」と出れば成功です。
試合が終わっていれば matches.json にスコアが入り、Netlifyが自動デプロイします。

## 手動で結果を直したいとき

`matches.json` の該当行の末尾（10番目の要素）にスコア文字列を書くだけです。

```json
[6,12,"4:00",1,"A","メキシコ","南アフリカ","N4D",null,"2-1"]
```

PK戦は `"1-1 PK4-2"` の形式で書くと勝者を自動判定します。
（次回のActions実行でAPI側のデータと一致していればそのまま維持されます）

## 大会終了後

7/21以降はスクリプトが自動でスキップ動作になりますが、Actionsの実行自体を止めたい場合は
`.github/workflows/update-results.yml` を削除するか、Actionsタブからワークフローを Disable してください。
