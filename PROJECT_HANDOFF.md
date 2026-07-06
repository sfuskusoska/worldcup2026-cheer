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
1. **トーナメント表**（ブラケット）：R32〜決勝。**応援チームの持ち主表示**は「持ち主カラーの左バー（`box-shadow: inset 3px 0 0 var(--own)`）＋名前チップ（`.owner-tag`、その人の色に人物名の漢字）」（⚠️旧: 7px色ドット `.owner-dots` は「小さくて分かりにくい」というユーザー指摘で**廃止**）。さらに上部「強調表示」で自分を選ぶと、**自分の応援チームを含むカードが自分の色で発光**（`.match-card.mine-owner`、`--mecolor`）＋自分の行が色枠（`.team-row.mine`）＝縮小した全体表示でも自分のチームの位置が一目で分かる（`renderBracket`が`me`依存になったため`setMe`が`renderBracket`も呼ぶ）。試合が近いとタグ色分け（3日以内/24時間以内/ライブ/終了）。全体表示/スクロール表示/詳細表示の3モード。
   - **勝ち上がりの軌跡グロー**（`winnerConnectorSegments`/`drawConnectors`内）：勝者が確定した区間のコネクタ線を、**その勝者チームの持ち主カラー**（持ち主なしはアクセント色#14d4a5）でグロー表示（canvasのshadowBlur＋白コアの2重ストローク）。勝ち進むほど軌跡がチェーン状に伸び、誰の応援チームがどのルートで勝ち上がったか一目で分かる。通常版の機能（ベータ限定ではない）。検証：ピクセル実測でグロー区間と未確定区間の色差を確認、複数勝者のシミュレーションでチェーン・色分け・詳細表示モードでの動作を確認済み。
2. **応援ポイントのリーダーボード**（`renderLeaderboard`）。
3. **優勝オッズ表**（全31チーム、順位・オッズ・確率・予想pt・持ち主列）。
4. **遊びルーレット**（`openRoulette`〜`stopRoulette`）：canvas回転式、スタート/ストップ操作。**配分には一切反映しない、純粋なお楽しみ機能**として明示的に分離済み（過去に配分決定用ルーレットだった時期もあったが、ホスト手動配分方式に変更後は遊び専用に格下げ）。
5. **戦術ボード**（`FORMATIONS`〜`renderTactics`など、ページ最下部）：11vs11マグネット式フォーメーションボード。4-3-3/4-4-2/4-2-3-1/3-5-2切替、ドラッグ移動（Pointer Events、マウス/タッチ両対応）、リセット、保存（検討中・localStorageに最大20件スナップショット）。マグネットは`width:5.5%`（当初の半分）。**注意**：実際の得点シーンの選手座標データはアプリに一切ないため、標準フォーメームのテンプレート位置を初期配置として使っている（画面上にもその旨を注記済み）。
   - **選手名編集**：全マグネット分の入力欄（グリッド）は廃止済み。今は`TACTICS_STATE.selected`で選択中のマグネットを1つだけ管理し、下部の`#tacticsSingleEditor`（1本の入力欄）で編集する方式。マグネットをタップ（移動なしのpointerdown→pointerup、判定しきい値4px）すると選択、ドラッグ（4px超の移動）だと選択は変えず位置だけ動く（`attachMagnetDrag`内の`moved`フラグで判定）。フォーメーション切替後も選択IDは維持される。
   - **⚡ スタメン反映**（`reflectLineup`）：ESPN公開JSON（`site.api.espn.com/.../summary?event={id}`の`rosters[]`）から、選んだ試合（`#lineupMatchSelect`）の実際のスタメン11人×2チーム・フォーメーション・控え選手を自動反映。イベントIDは事前に持っていないため、キックオフ日の`scoreboard?dates=YYYYMMDD`を叩いて`teamIdFromEspn()`でチーム一致するイベントを逆引き（`findEspnEventId`）。試合サマリーは`fetchMatchSummary(matchId)`＋`MATCH_SUMMARY_CACHE`で共有・キャッシュ（ゴール位置マーカーも同じキャッシュを使う）。控えは`#tacticsBench`にチップ表示（`renderTacticsBench`）。**キックオフ約1時間前より前は`roster[]`が空**なので「スタメンはまだ発表されていません」とtoastして終了する（クラッシュしない）。実データで動作確認済み（M73＝南アフリカ vs カナダで11人×2＋控え28人・フォーメーション4-4-2まで反映成功）。
   - **チームA/B独立フォーメーション**：`TACTICS_STATE`は`formationA`/`formationB`を別々に持つ（旧`formation`単一フィールドは廃止）。プリセット4種（4-3-3/4-4-2/4-2-3-1/3-5-2）にない実フォーメーション文字列（例"3-4-2-1"）は`autoLayoutFromFormation()`が`-`区切りの数字列からライン状の座標を自動生成し、`layoutFor()`が「プリセットにあればそれ、なければ自動生成、どちらも無理なら4-3-3」の順で解決する。`reflectLineup`は home→`formationA`、away→`formationB`に別々に代入し、`#formationAutoNote`に「A: x / B: y」を表示。**実データで検証済み**：ブラジル×日本戦（ESPN event 760487）でブラジル=4-3-3（プリセット）・日本=3-4-2-1（プリセット外・自動生成）が同時に正しく反映されることを確認。手動で`#formationSelect`を変更した場合はA/B両方が同じプリセットに揃う（従来通りの一括変更、意図的な仕様）。
   - **⚽ ゴール位置マーカー**（`initGoalMarkers`/`showGoalMarker`）：ESPNの`summary`内`keyEvents[].scoringPlay`（`fieldPositionX`/`fieldPositionY`、得点者、時刻）を使い、得点シーンをピッチ上に黄色いドット＋ラベルで1点だけ表示する機能。`#goalEventSelect`で得点を選び「⚽ ゴール位置」ボタンで表示。**22人の実配置ではなく、得点者1名の近似的な目印**（同じ注意書きが必要）。座標変換は`goalMagnetPosition(side, fx, fy)`：チームA/Bで`fieldPositionX/Y`の意味が逆になるため`teamSideForEspnTeamId()`でどちらのチームの得点かを判定してから変換する。**実データで両チーム分の対称性を検証済み**（日本の得点=下側/自陣寄り表示、ブラジルの得点=上側/相手ゴール手前表示、いずれもゴール前の妥当な位置）。得点なし試合・未反映のまま押した場合はtoastのみでクラッシュしない。
   - **控え⇄スタメンの入れ替え**（`swapWithBench`/`attachBenchChipDrag`）：`#tacticsBench`の控えチップをスタメンのマグネットへ**ドラッグ＆ドロップ**すると入れ替わる（Pointer Events、マウス/タッチ両対応。マグネットのドラッグ移動と実装パターンは同じだが別関数）。ドラッグ中は`document.elementFromPoint()`で真下のマグネットを判定し、同じチーム（`a`/`b`）の枠だけ`.drop-target`でハイライト、異なるチームへドロップした場合は入れ替えずtoastで案内。入れ替えると、出ていくスタメンの名前・背番号がその控え枠に戻る（元々空スロットだった場合は控えに戻さず単に消費）。状態は`TACTICS_STATE.names`と並行して`TACTICS_STATE.starterJersey`（スタメンの背番号、`wc2026TacticsJersey`に保存）・`TACTICS_STATE.benches`（控え一覧、`wc2026TacticsBenches`に保存）で管理し、リロード後も入れ替え結果が保持される。**実データで検証済み**（ブラジル×日本戦でNeymar(控え)⇄M. Cunha(スタメンa-9)の入れ替え、クロスチーム拒否、リロード後の保持、空スロットへの入れ替えすべて確認）。
   - **マグネットの背番号表示**（`magnetNumberLabel`）：`.m-num`は常に実際の背番号（`TACTICS_STATE.starterJersey`）を優先表示し、未反映時のみフォーメーション枠番号にフォールバックする。⚠️過去に「入れ替えても番号が変わらない」バグがあった（`.m-num`が枠番号を表示するだけで`starterJersey`を一切参照していなかった）。修正済み・検証済み（Neymar(10)⇄M.Cunha(9)入れ替えで表示が9→10に変化、リロード後も保持）。
   - **ピッチのスクロール**：`.pitch-wrap`に`touch-action:none`を付けない。ドラッグ対象（`.magnet`/`.bench-chip`）個別にのみ`touch-action:none`を付与する。⚠️過去にピッチ全体へ`touch-action:none`を付けていたため、芝生部分に触れただけでもページスクロールがブロックされていた（修正済み）。
   - **矢印・ライン描画**（`toggleDrawMode`/`attachArrowDrawing`/`renderArrows`/`clearArrows`）：「✏️ 矢印を描く」で描画モードON（`#pitchWrap`に`.draw-mode`クラス、CSSで`#magnetLayer`の`pointer-events:none`にしてマグネット操作と競合しないようにする）。ピッチ上をドラッグすると矢印（SVG`<line>`+矢じり`marker-end`）を1本追加。`#arrowLayer`（`viewBox="0 0 68 105"`、pitch座標系と共通）に描画。`TACTICS_STATE.arrows`配列を`wc2026TacticsArrows`にlocalStorage保存、リロードで復元。「🧹 矢印を消す」で全消去。
   - **アニメーション（フレーム記録・再生）**（`recordFrame`/`playFrames`/`jumpToFrame`/`deleteLastFrame`）：「📍 フレーム記録」で現在のマグネット配置を1フレームとして`TACTICS_STATE.frames`に保存（`wc2026TacticsFrames`）。「▶️ 再生」でフレームを順番に適用し、CSSトランジション（`#magnetLayer.animating .magnet { transition: left 1s ease, top 1s ease; }`）で滑らかに移動する。**重要**：`renderTactics()`の毎回`innerHTML`再構築ではなく、`applyPositionsToDom()`で既存DOM要素の`style.left/top`だけを書き換えることでトランジションを効かせている（innerHTML再生成だと要素が作り直されて瞬間移動になる）。フレームチップ（F1, F2…）をクリックでそのフレームへジャンプ。
   - **Undo/Redo**（`pushHistory`/`undoTactics`/`redoTactics`）：`positions`・`names`・`arrows`をJSON文字列でスタック保存（最大30件）。呼び出し箇所：マグネットのドラッグ開始時（`attachMagnetDrag`のpointerdown冒頭）、名前欄フォーカス時（`oninput`ではなく`onfocus`で1回だけ）、リセット時。矢印描画確定時にも呼ぶ。ドラッグ移動・名前編集・矢印追加それぞれでundo/redoが正しく機能することを検証済み。
   - **セットプレーテンプレート**（`SET_PIECE_PRESETS`/`applySetPiece`）：コーナー攻撃/守備・FKサイドの3種。**座標はすべて叩き台**で、実戦のセオリーそのものではなく見た目を見ながら調整する前提の初期値（コメントで明記）。チームBに適用する場合は`68-x, 105-y`で鏡映。
   - **PNG書き出し**（`exportTacticsPng`）：`html2canvas`（CDN読込）で`.tactics-section`全体を撮影しPNGダウンロード。検証済み（実際にPNGのdata URLが生成されることを確認）。
   - **国旗表示**（Tier2、`TACTICS_STATE.teamFlags`）：`reflectLineup`実行時に home/away のESPNチームを`teamIdFromEspn()`で内部コードへ変換して記録し、各マグネットの右下に小さい国旗を表示。実データ（ブラジル×日本）で両チーム11人ずつに国旗が出ることを確認済み。
   - **GIF書き出し**（Tier2、`exportFramesAsGif`）：`gif.js`（CDN読込）で記録済みフレームを順番にレンダリングし、アニメーションGIFとしてダウンロード。**重い処理**（フレーム数×約2秒程度、3フレームで実測7秒。ボタン付近に注記を表示）。⚠️**修正した不具合**：`gif.js`のworkerスクリプトをCDNの絶対URLのまま`new GIF({ workerScript: '...' })`に渡すと、ブラウザの同一オリジン制約でWorker生成が失敗する（`Failed to construct 'Worker'`）。`getGifWorkerScriptUrl()`でworkerスクリプトを一度`fetch`し、`Blob`化して`URL.createObjectURL`したBlob URLを`workerScript`に渡すことで回避（実際に有効なGIF89aバイナリが生成されることを確認済み）。
   - **複数プレーのタブ管理**（`saveTacticsSnapshot`/`loadTacticsSnapshot`/`renderTacticsSaveList`）：保存内容に`formationA/B`・`arrows`・`frames`・`label`（「プレー1」「プレー2」…）を含める。タブ風の横並び表示（`.tactics-save-row`を`inline-flex`）。⚠️**修正した不具合**：スナップショットIDに`Date.now()`のみを使っていたため、同一ミリ秒内に連続保存するとID衝突し、読込/削除ボタンが誤ったスナップショットを操作するバグがあった。`nextTacticsSnapshotId(list)`で既存リストと衝突しないIDを保証するよう修正済み（衝突しないことを検証済み）。
6. **演出（旧ベータ版モード）**（`fireConfetti`/`startBetaCountdown`ほか）：⚠️**旧「🧪 ベータ版」トグルは廃止し、演出を通常版に統合（常時ON）**（ユーザー指示）。`isBeta()`は`return true`固定、`initBeta()`が`body.beta-mode`を常時付与＋`startBetaCountdown()`起動、トップバーの`#betaBtn`は削除。`applyBeta`/`toggleBeta`は未使用のまま残置。演出は引き続き`body.beta-mode`配下CSS＋`isBeta()`ガードに閉じ込めてあり、`prefers-reduced-motion`も尊重。常時ONの演出：紙吹雪（切替時・遊びルーレット決定時・ゴール位置表示時。canvasを都度生成し約2.6秒で自動削除）、リーダーボード首位に👑＋金色グロー（CSS疑似要素）、リーダーボード行のスライドイン、動くオーロラ背景（`body::before`固定レイヤー）、カードのホバーリフト、ピッチのスタジアム照明風グロー、**次戦までのリアルタイムカウントダウン**（heroのstatus-stripに1秒更新のpillを動的追加、`nextUpcomingMatch()`が未消化試合の最短キックオフを算出）。アニメーション系はすべて`prefers-reduced-motion`を尊重（reduce時は紙吹雪もスキップ）。⚠️Claude Previewはreduced-motionエミュレート環境なので、紙吹雪の見た目確認は実機で行うこと（エンジン自体の動作はガードをバイパスして検証済み）。
7. **BGM**（`startBgm`/`stopBgm`）：`assets/audio/pre-match-bgm.mp3`（曲名: Arena of Champions Loop）を2プレイヤーでクロスフェードループ再生。iOS対策で、タップ操作内に控えプレイヤーも無音で解錠する処理あり。**音源ファイルは必ずgit管理下に置くこと**（一度untrackedのままpushし忘れてスマホで404→無音になった事故あり）。
8. **イブラヒモビッチ・トリビュート**（ページ最上部）：日本×ブラジル戦（1-2）後のコメント引用。閉じるとlocalStorageに記憶。**「フェイクニュースの可能性が高い」という注意書きを本人発言の裏取りができないため付与済み**（ユーザー指示）。
9. **スコア更新**（`refreshScores`）：ESPN公開JSON（`site.api.espn.com`）を叩いて試合結果を反映。CORS失敗時はallorigins.winプロキシにフォールバック。
10. **個人タイトル争い＝得点王・アシスト王ランキング**（`loadLeaders`/`renderLeadersList`/`resolveAthlete`）：ボタン押下でオンデマンド取得。ソースは**ESPN core API**の`https://sports.core.api.espn.com/v2/sports/soccer/leagues/fifa.world/seasons/2026/types/{1..3}/leaders`（`goalsLeaders`/`assistsLeaders`カテゴリ。`types/1`にデータあり。空なら2→3をフォールバック試行）。各leaderは`value`＋`athlete.$ref`＋`team.$ref`のみ（名前はインラインに無い）。**選手名は`athlete.$ref`を1件ずつ解決**（`resolveMany`で並列度6、`ATHLETE_CACHE`でキャッシュ・得点/アシスト重複選手を共有）。⚠️`$ref`は`http:`なのでhttpsページで**mixed contentになる→`replace(/^http:/,'https:')`必須**。**所属国は追加フェッチせず`athlete.flag.href`（例`.../countries/500/fra.png`）の3レターコードを`espnTeamCodeFromFlag`で抜き出し`ALIASES`で内部コードへ変換**（=フェッチ数を選手分だけに抑制）。内部コードから`ownerOf()`で「誰の応援チームの選手か」を色バッジ表示、`me`の選手は`.ldr.mine`で色枠（`setMe`→`renderLeadersOwners`で再描画）。⚠️flagコードが内部と一致しない選手（例: Brahim Díaz）は持ち主未表示になるが名前・数値は出る（グレースフル）。各上位10人。**実データで検証済み**（Mbappé/Haaland/Messi等が正しい所属国＋持ち主で表示）。
11. **My応援チームの試合スケジュール**（`renderMyTeamSchedule`/`upcomingMatchIdForTeam`/`tickCountdowns`）：上部「強調表示」で選んだ人（`me`）の応援6チームについて、各チームの**次の未消化試合**（対戦相手・ラウンド・JST）と**キックオフまでの1秒カウントダウン**（`.cd-timer[data-utc]`を`setInterval(tickCountdowns,1000)`で更新）を並べる。敗退チームは「❌ 敗退」、優勝は「🏆 優勝」、キックオフ過ぎ未確定は「結果待ち」。`me`未選択時は案内文。
12. **応援チーム別 得点/失点サマリー**（`renderCheerGoalsSummary`/`teamGoalStats`）：5人それぞれの応援6チームについて、決着済み試合（`m.score`あり）の累計**得点/失点/得失点差**を集計し、得点降順で緑（得点）・桃（失点）の2本バーで比較表示。

## 依存ライブラリ（CDN）
- `html2canvas` 1.4.1（PNG/GIF書き出しで使用）
- `gif.js` 0.2.0（GIF書き出しで使用。workerスクリプトはCORS制約のためBlob URL経由で読み込む。上記「GIF書き出し」の注意点を参照）
- 両方とも`</body>`直前、既存アプリの`<script>`より前に`<script src>`で読込。

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
- **`document.elementFromPoint()`はビューポート内座標しか検出できない**：戦術ボードはページ下部にあるため、`getBoundingClientRect()`の値（スクロール位置に応じたビューポート相対座標）が`window.innerHeight`を超えることがある。その状態で`elementFromPoint`を呼ぶと常に`null`が返る（要素が実在しないのではなく、単に画面外にあるため検出できないだけ）。ドラッグ&ドロップ系（`attachBenchChipDrag`など）を`preview_eval`で検証するときは、必ず対象要素を`scrollIntoView()`してからクリック/ドラッグ座標を計算すること。

## 直近のコミット履歴（新しい順）
```
(次コミット) 得点王・アシスト王ランキング / My応援チームの試合スケジュール / 応援チーム別 得点・失点サマリー を追加；旧ベータ演出を通常版に統合(常時ON・🧪トグル廃止)；ブラケットの持ち主表示を7px色ドット→名前チップ＋持ち主カラー左バー＋自分スポットライトに刷新；grid-2パネルのモバイル横クリップ(オッズ表)を修正
f6f6ea8 Glow winner-advancement paths on the bracket connectors
652213e Add opt-in beta mode with enhanced visuals and effects
3c0874e Expand tactics board: arrows, animation, undo/redo, set pieces, PNG/GIF export, flags, multi-play tabs
d432db1 Fix jersey number not updating on bench swap; fix page-scroll lock
0f4ac06 Update PROJECT_HANDOFF.md commit history with bench-swap hash
1ad9efd Add bench<->starter swap via drag-and-drop on the tactics board
7c7e44c Independent per-team formations + auto-layout; goal position marker
3c55dc5 Update PROJECT_HANDOFF.md commit history after lineup-reflect feature
bc6e2aa Add ESPN lineup auto-reflect + bench, single-select magnet name editor
a6ac46d Move tactics board to bottom; halve magnet size; harden drag capture
10d1082 Add PROJECT_HANDOFF.md for quick context loading in a new session
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
