# Supabase セットアップ手順

全員（5人）の担当チームを共有・同期するための無料DBを用意します。所要 約5分。

## 1. プロジェクト作成
1. https://supabase.com を開き、**Sign in with GitHub** でログイン
2. **New project** を作成
   - Name: 任意（例 `worldcup2026`）
   - Database Password: 任意（控えておく / このアプリでは使いません）
   - Region: `Northeast Asia (Tokyo)` など近い場所
3. 作成完了まで 1〜2分待つ

## 2. テーブル作成（SQLを実行）
左メニュー **SQL Editor** → 新規クエリに以下を貼り付けて **Run**。

```sql
-- 5人の担当チームを保存するテーブル
create table if not exists public.wc2026_picks (
  user_id    text primary key,
  team       text not null,
  method     text default 'manual',
  updated_at timestamptz default now()
);

-- RLS（行レベルセキュリティ）を有効化し、anon に読み書きを許可
alter table public.wc2026_picks enable row level security;

create policy "anon read"   on public.wc2026_picks for select using (true);
create policy "anon insert" on public.wc2026_picks for insert with check (true);
create policy "anon update" on public.wc2026_picks for update using (true) with check (true);
create policy "anon delete" on public.wc2026_picks for delete using (true);

-- リアルタイム反映を有効化（任意・推奨）
alter publication supabase_realtime add table public.wc2026_picks;
```

> ※ これは「URLを知っている5人で使う」前提のオープン設定です。誰でも読み書き可能なので、
> 公開範囲が広がる用途には向きません（今回の身内プールでは問題ありません）。

## 3. URL と anon キーを取得
左メニュー **Project Settings（歯車）** → **API**
- **Project URL**（`https://xxxx.supabase.co`）
- **Project API keys** の **`anon` `public`** キー（`ey...` で始まる長い文字列）

この2つを Claude に貼り付けてください。`index.html` に差し込みます。

```js
const SUPABASE_URL = "https://xxxx.supabase.co";
const SUPABASE_ANON_KEY = "ey...";
```

設定後はページを開くだけで、5人の選択が全員にリアルタイム共有されます。
