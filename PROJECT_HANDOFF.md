# W杯2026 応援ゲーム — 引き継ぎ資料

新しいセッションはこの1ファイルを読めば全体像を把握できます。コード自体（`index.html`、単一ファイル・約1700行）が正なので、詳細確認は必ずコードを読んでから。

## これは何か
FIFAワールドカップ2026 ノックアウト表＋優勝オッズに、「5人が応援チームを持って勝ち上がりを競う」ゲーム機能を足した、GitHub Pages公開の1ファイルWebアプリ。

- **公開URL**: https://sfuskusoska.github.io/worldcup2026-cheer/
- **リポジトリ**: https://github.com/sfuskusoska/worldcup2026-cheer （GitHubアカウント: sfuskusoska、`gh` CLI認証済み）
- **ローカルパス**: `D:\Claude\code_projects\worldcup2026-cheer\index.html`
- **ローカル確認**: `.claude/launch.json` の `"wcpool"` 設定 → `localhost:5599`（Claude Previewツールで起動）
- **元データ**: `C:\Users\s.fukuoka\Downloads\worldcup_knockout_odds_bracket_jp_v2.html`（最初のベースHTML）

## 名称について（重要・地雷）
アプリ名は必ず **「W杯2026 応援ゲーム」**。「プール」「予想」という単語は**禁止**（ユーザーが「賭けっぽい」という理由で明示的に拒否）。旧リポジトリ名 `worldcup2026-pool` は改名済みで `worldcup2026-cheer` にリダイレクトされる。

## 5人の固定ユーザー
`index.html` 内 `USERS` 配列：岡・大・田・片・南（id: oka, dai, ta, kata, minami）。増減や名前変更はここを直す。

## 配分の仕組み（何度も壊れて直した最重要ロジック）
- 30チーム（終了済みのM73＝南アフリカ対カナダを除く）を、**5人へ各6チームずつ、ホスト（ユーザー本人）が手動で確定**。リアルタイム共有DBは**使わない**方針（当初Supabase案があったが撤回）。
- 確定配分は `DEFAULT_ASSIGNMENTS` にハードコードで焼き込み済み（下記が現在の確定内容）：
  - 岡: FRA, NOR, BEL, CIV, MEX, AUS
  - 大: NED, CRO, ESP, BIH, SEN, EGY
  - 田: JPN, BRA, GER, SWE, CPV, GHA
  - 片: PAR, MAR, AUT, ARG, DZA, COL
  - 南: CAN, POR, USA, ECU, ENG, SUI
  - 未配分: COD（DRコンゴ）
- 画面の「✏️ 配分を編集」でホストが自分でチーム追加/削除できる（重複不可・各6チーム上限・終了済み除外）。
- **保存の仕組み**：`ASSIGNMENTS` を編集するたびに (1) `localStorage`（キー `wc2026Assign2`）と (2) URLハッシュ `#p=...`（`encodeURIComponent(JSON.stringify(...))`）の両方に保存。読み込み時は **URLハッシュ最優先→なければlocalStorage→なければDEFAULT_ASSIGNMENTS**。
  - ⚠️ **過去にここで2回バグを出した**：①保存時にURLハッシュを更新し忘れて「古い共有リンクを開くと編集前に戻る」→修正。②`btoa`のbase64に`+`が含まれるとURL解析で空白化けして復元失敗＋プライベートブラウズでlocalStorageが効かない→`encodeURIComponent`方式に変更し、保存キー/パラメータ名も `a`→`p`, `wc2026Assign`→`wc2026Assign2` にバージョンを上げて**古い壊れたデータを強制的に無視**させて解決。
  - この経緯があるので、配分保存周りを触るときは必ず「localStorageを空にした状態」と「壊れた/古いハッシュを付けた状態」の両方でリロード検証すること。

## 得点ルール
- 1勝＝1ポイント（`winCount()`で試合の`winner`をカウント）。
- **例外**：M73（カナダ×南アフリカ、配分確定前に決着済み）はポイント対象外。`NOSCORE_MATCH_IDS = new Set(['m73'])` で除外。カナダの今後の勝利は通常どおり加算。
- 優勝5点・準優勝4点・3位決定戦の勝者も4点（同じ勝利数になるよう試合構成されている）。

## 予想ポイント（期待値）
- `EXPECTED_WINS` / `ODDS` 配列に、Elo実力値×ブラケット構造を50万回モンテカルロシミュレーションした「各チームの期待勝利数」を静的に埋め込み済み（Node.jsスクリプトで別途計算し、結果だけ貼り付けている。アプリ内で毎回計算はしていない）。
- `userPredicted(userId)` = そのユーザーの担当6チームの期待勝利数の合計。カード・リーダーボードに「予想 X.X pt」で表示。
- 全31生存チームの合計期待勝利数 ≒ 31点（総試合数と一致）になるよう設計。検算済み。

## 主要機能一覧
1. **トーナメント表**（ブラケット）：R32〜決勝、担当ユーザーの色ドット表示、試合が近いとタグ色分け（3日以内/24時間以内/ライブ/終了）。全体表示/スクロール表示/詳細表示の3モード。
2. **応援ポイントのリーダーボード**（`renderLeaderboard`）。
3. **優勝オッズ表**（全31チーム、順位・オッズ・確率・予想pt・持ち主列）。
4. **遊びルーレット**（`openRoulette`〜`stopRoulette`）：canvas回転式、スタート/ストップ操作。**配分には一切反映しない、純粋なお楽しみ機能**として明示的に分離済み（過去に配分決定用ルーレットだった時期もあったが、ホスト手動配分方式に変更後は遊び専用に格下げ）。
5. **戦術ボード**（`FORMATIONS`〜`renderTactics`など、ページ最下部）：11vs11マグネット式フォーメーションボード。4-3-3/4-4-2/4-2-3-1/3-5-2切替、ドラッグ移動（Pointer Events、マウス/タッチ両対応）、リセット、保存（検討中・localStorageに最大20件スナップショット）。マグネットは`width:5.5%`（当初の半分）。**注意**：実際の得点シーンの選手座標データはアプリに一切ないため、標準フォーメームのテンプレート位置を初期配置として使っている（画面上にもその旨を注記済み）。
   - **選手名編集**：全マグネット分の入力欄（グリッド）は廃止済み。今は`TACTICS_STATE.selected`で選択中のマグネットを1つだけ管理し、下部の`#tacticsSingleEditor`（1本の入力欄）で編集する方式。マグネットをタップ（移動なしのpointerdown→pointerup、判定しきい値4px）すると選択、ドラッグ（4px超の移動）だと選択は変えず位置だけ動く（`attachMagnetDrag`内の`moved`フラグで判定）。フォーメーション切替後も選択IDは維持される。
   - **⚡ スタメン反映**（`reflectLineup`）：ESPN公開JSON（`site.api.espn.com/.../summary?event={id}`の`rosters[]`）から、選んだ試合（`#lineupMatchSelect`）の実際のスタメン11人×2チーム・フォーメーション・控え選手を自動反映。イベントIDは事前に持っていないため、キックオフ日の`scoreboard?dates=YYYYMMDD`を叩いて`teamIdFromEspn()`でチーム一致するイベントを逆引き（`findEspnEventId`）。控えは`#tacticsBench`にチップ表示（`renderTacticsBench`）。**キックオフ約1時間前より前は`roster[]`が空**なので「スタメンはまだ発表されていません」とtoastして終了する（クラッシュしない）。実データで動作確認済み（M73＝南アフリカ vs カナダで11人×2＋控え28人・フォーメーション4-4-2まで反映成功）。
6. **BGM**（`startBgm`/`stopBgm`）：`assets/audio/pre-match-bgm.mp3`（曲名: Arena of Champions Loop）を2プレイヤーでクロスフェードループ再生。iOS対策で、タップ操作内に控えプレイヤーも無音で解錠する処理あり。**音源ファイルは必ずgit管理下に置くこと**（一度untrackedのままpushし忘れてスマホで404→無音になった事故あり）。
7. **イブラヒモビッチ・トリビュート**（ページ最上部）：日本×ブラジル戦（1-2）後のコメント引用。閉じるとlocalStorageに記憶。**「フェイクニュースの可能性が高い」という注意書きを本人発言の裏取りができないため付与済み**（ユーザー指示）。
8. **スコア更新**（`refreshScores`）：ESPN公開JSON（`site.api.espn.com`）を叩いて試合結果を反映。CORS失敗時はallorigins.winプロキシにフォールバック。

## 画像・音源アセット
- `assets/image/top.png`：トップのブランド画像（ユーザー提供）。
- `assets/audio/pre-match-bgm.mp3`：BGM（ユーザー提供、Arena of Champions Loop）。
- **両方ともgit管理下にあることを確認してからpushする**（過去に忘れてライブで404になった）。

## デプロイ手順（毎回同じ）
```
cd D:\Claude\code_projects\worldcup2026-cheer
git add -A
git commit -F <一時ファイル>.txt   # PowerShellでは -m の直書きだと日本語+改行+ダブルクォートで壊れるため、UTF8Encoding($false)で書いたtxtファイルを-Fで渡す
git push
```
push後、GitHub Pagesのビルド完了は `gh api repos/sfuskusoska/worldcup2026-cheer/pages/builds/latest` をポーリングして確認。ビルド後、ライブHTMLに変更点のマーカー文字列が含まれるかを`Invoke-WebRequest`で検証してから「反映済み」と報告する運用にしている（自己申告で終わらせない）。

## 検証の作法
- Claude Previewツール（`preview_start`/`preview_eval`/`preview_console_logs`）でローカル起動し、`preview_eval`でJS関数を直接叩いて状態を検証するのが基本。
- `preview_screenshot`は本環境で頻繁にタイムアウトする（既知の不安定さ）。失敗したら無理に使わず、DOM実測（`getBoundingClientRect`など）で代替する。
- 配分の永続化バグのように「見た目は動いてもリロードやプライベートブラウズで壊れる」系の不具合があるため、**保存系の機能は必ずリロードを挟んだ検証をする**。

## 未確定・保留事項
- Supabase等の外部共有DBは**使わない方針で確定**（ホストが手動で配分してURL/localStorageに焼き込む方式に変更済み）。関連の残骸（`SUPABASE_SETUP.md`等）は既に削除済みのはず。
- 戦術ボードの「保存」機能は明示的に「検討中」表記。仕様（誰と共有するか、試合ごとに複数持つか等）は未確定。
- COD（DRコンゴ）が誰にも配分されていない（意図的な余り）。
- **プレビュー環境の既知の不具合**：`preview_start`直後や長時間経過後に`window.innerWidth`等が0になり、`getBoundingClientRect()`の値が壊れることがある（今回のドラッグ検証中に発生）。`preview_resize`にpreset名を渡しても直らないことがあり、その場合は明示的な`width`/`height`数値（例1280x900）を指定するか、サーバーを`preview_stop`→`preview_start`し直すと復旧する。ドラッグ/座標系の検証で数値が不自然（極端に大きい/0）なときはまずこれを疑う。

## 直近のコミット履歴（新しい順）
```
f6330e8 Add tactics board; fake-news disclaimer on tribute
09a059d Add Ibrahimovic tribute to Japan at the top of the page
e4b1b65 Drop stale broken state via storage version bump (#a->#p, key bump)
951dd0c Fix selection reset in Private Browsing; bake host's allocation as default
199f291 Exclude Canada's pre-draft opener (M73) from points
65b9413 Fix: edits reverted on reload (stale URL hash overrode localStorage)
b784021 Harden BGM playback for iOS; sync docs
5e6f9f9 Top image, full-team odds + predicted points, Canada assignable; ship assets
479ace4 Add host team-selection editor (no backend)
e8c03fc Rebrand to W杯2026 応援ゲーム (repo worldcup2026-cheer)
271165b Replace roulette with spinning wheel; enforce no-duplicate picks
81ea8f8 Fix bracket card spacing on mobile
734dc1d Add World Cup 2026 knockout pool (5-user shared picks)
```
