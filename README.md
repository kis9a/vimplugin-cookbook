# vimplugin-cookbook v2

## 概要

このドキュメントは、Vim / Neovim 両対応プラグインを「長く壊れにくく」開発・保守するための共通ガイドです。
責務ごとに層を分けたアーキテクチャ、戻り値と変更操作の共通規約、永続化・非同期・専用バッファの実装パターン、テストと検証の手順、互換性メモを整理し、実装のブレ・互換性事故・データ消失・起動や実行の性能劣化を防ぐことを目的とします。

対象は「状態を持ち、専用バッファで表示し、外部コマンドや非同期処理を使う」規模のプラグインです。数十行のユーティリティには重すぎるので、その場合は層のうち必要なものだけを採ります。

## 前提

Vim / Neovim 両対応プラグインとして実装する

### ルール

* 基本は レガシー Vimscript（Vim9script・Lua 依存は禁止。片方のエディタで動かない）
* バージョン判定より 機能検出：`has`, `exists('*fn')`, `executable()`、または実際に 1 回試して `try` で捕まえる
* Vim/Neovim 固有 API（job/channel/UI/通知/時刻）は compat 層に隔離し、呼び出し側は共通 API のみ使用
* **外部依存（外部コマンド / 他プラグイン / ネットワーク / ファイル）** は deps 層と storage 層に隔離し、呼び出し側は共通 API のみ使用
* 他プラグインの関数は、既定では実行時に呼ばない。純粋関数ならライセンス表示を保ったうえで移植する。呼ぶなら deps に隔離し、未導入・未ロード時の代替を決める（遅延ロード中のプラグインの autoload 関数は E117 になるため、`exists('*other#fn')` では「未ロード」と「未インストール」を区別できない）
* 機能が無い環境向けのフォールバックは用意する。ただし **ネットワーク通信は同期にフォールバックしない**（取得を省略し、利用者の明示操作で再試行させる）
* `plugin/` は定義と軽量な設定だけ（`command!`、既定値の `let`、ロードガード、`cpoptions` の保存復元）。I/O・ジョブ起動・描画・autoload を読み込ませる呼び出しを置かない
* グローバルな autocmd は、理由を説明できるときだけ張る。基本は自分のバッファに付く `<buffer>` 付きの autocmd と `ColorScheme`。性質上グローバルになるもの（`VimEnter`, `DirChanged`, `FocusLost` 等）は augroup・対象の絞り込み・低負荷・解除手段を揃えて help に書く
* 構成は標準 runtime（`plugin/`, `autoload/`, `doc/`）。名前空間で汚染を避ける
* 関数名は snake_case。内部関数は `s:`。公開 API は `autoload/${plugin_name}.vim` に集約。すべての関数に `abort`
* 非ASCII（日本語メッセージ等）を含むなら `scriptencoding utf-8` を明示。`\` 行継続や `<SID>` を使うファイルでは `cpoptions` ガードを置く（置くなら復元まで必ず書く）
* ネットワーク通信は利用者が止められること（`g:${plugin_name}_xxx = 0`）。既定 ON にするなら help の Privacy 節に何を・いつ・どこへ送るかを書く
* 対応する最低版を宣言する。「手元の版で動いた」は保証ではない。使う機能ごとに必要なパッチを調べ、`Vim >= X.Y.patch` / `Neovim >= A.B` と必要な外部コマンドを README と help に書き、その下限でもテストを回す。下限で確認できない機能は実行時に判定して代替経路を持つ

### 参照許可

* Vim/Neovim の公式 help を根拠にしてよい（context7 参照も可能）
* context7 / serena mcp の利用を許可

### 出力

* 実装（差分が分かる形）
* 互換性メモ（分岐点と理由、フォールバック）
* Vim/Neovim それぞれの簡単なテスト手順
* 設計からの逸脱とその理由

### レガシー Vimscript を使う理由

* 両対応の「共通土台」として最も壊れにくい
* Vim/Neovim の差分が出る箇所（job/channel/UI 等）は **薄い compat 層**で吸収できる
* Vim9script は Neovim 非対応、Lua は Vim 非対応のため、両対応を最優先する限り避けるのが安全

---------

## 確認観点（レビュー/自己点検チェックリスト）

### パフォーマンス劣化懸念の払拭

* [ ] 起動時に重い処理をしていない（`plugin/` は定義と軽量な設定だけ。`vim --startuptime` で測り、基準環境と測定値を記録する。1 ms 未満は参考値）
* [ ] グローバル autocmd が `ColorScheme` と、help に理由を書いたものだけ（`grep -rn autocmd autoload plugin` で `<buffer>` 無しを確認）
* [ ] タイマーのポーリングが無い。ジョブは必要なときだけ起動し、終わったら消える
* [ ] 想定上限（例: 1,000 件）のフィクスチャで描画・保存・編集の時間を `reltime()` で測り、基準環境と測定値を記録した（目標値は参考: 描画 100 ms 未満）。二乗の経路（重複検査・親探索・差分）は索引（Dictionary）を使っている
* [ ] コールバックから同期で呼ぶ関数（HTML 解析など）は入力長に対して線形

### セキュリティチェック

* [ ] `system()` / `:execute` / `:normal!` / `printf` の書式に **利用者入力を直結しない**
* [ ] 外部コマンドは argv List で渡す（文字列連結しない。List 非対応の版だけ `shellescape()` 連結）
* [ ] 外部コマンドの argv・環境変数・ログ・`:messages` に秘匿情報（トークン、パスワード）を出さない。出す情報と伏せる情報を分類している
* [ ] 外部コマンドは deps 層に隔離し、許可するコマンドが説明できる
* [ ] URL など外部に渡す値は許可する文字集合で検証している（保存時と使用直前の 2 回）
* [ ] HTTP は許可スキーム・リダイレクト上限・タイムアウト・応答サイズ上限・無効化オプションを持つ。任意 URL を受け付けるなら、ローカルホストやプライベートアドレスへ到達し得ること（SSRF）を受容するか制限するか決めて help に書く
* [ ] 利用者入力の文字列は制御文字・双方向制御文字を無害化し、UTF-8 として妥当なものだけ保存する
* [ ] 勝手にネットワーク通信しない（するなら opt-out を用意し Privacy 節に明記）

### データ保全

* [ ] 読めないファイル・ディレクトリ・権限 000・FIFO・シンボリックリンクのデータファイルで既存データが消えない（実際に試した）
* [ ] 壊れたファイル・上位スキーマのファイルには一切書かない
* [ ] 保存は一時ファイル + `rename()`。一時ファイルは権限を落としてから内容を書く。失敗時に一時ファイルが残らない
* [ ] 読む前にサイズ上限を見る（巨大ファイルを `readfile()` してから検証しない）
* [ ] 1 MiB の整形式入力・閉じていないタグ × 10,000・無限に出力するコマンドで固まらない（時間を測った）
* [ ] 片方のエディタで保存したファイルをもう片方が読める（不正 UTF-8・結合文字の扱いの差）

### 権限/影響範囲が強すぎないか

* [ ] `:command` はプレフィックス付き（衝突しない）
* [ ] `autocmd` は group を作り、解除/更新しやすい
* [ ] グローバル変数汚染をしていない（`g:${plugin_name}_...` のみ）
* [ ] 利用者の実際の vimrc で既定キーが動く（接頭辞マップ、`<Leader>`、`cnoremap`）。閉じる・開く・ヘルプのキーは常に素のキー

### 公開関数と層の線引き（API設計）

* [ ] 公開 API は `autoload/${plugin_name}.vim` に集約
* [ ] 層の依存表に無い呼び出しが無い（`grep -rn '${plugin_name}#' autoload`。動的呼び出しは目視）
* [ ] domain 層が `g:`・時刻・バッファ・ファイル・外部コマンドを使っていない
* [ ] README と help の「Commands / Keys / Functions / Options」と実体・既定値が一致
* [ ] 他プラグインの関数を呼ぶ箇所は deps 層の中だけで、未導入・未ロード時の動作が決まっている

### 互換性・配布

* [ ] `make test-vim` と `make test-nvim` が通る（エディタ固有で skip したテストは理由を記録）。`make lint` がエラー 0
* [ ] README と help に最低版（Vim はパッチ番号を含む）と必要な外部コマンドがあり、その下限で確認した。本体の対応 OS と、開発ツールチェーン（Makefile、pty E2E）の対応 OS を分けて書いた
* [ ] help に Privacy と Limitations の節がある
* [ ] 移植したコードの著作権表示とライセンス文を保っている
* [ ] 設計書（要件定義）に「実装結果と逸脱」を追記した

---------

## プラグインのアーキテクチャ

### 全体方針

* **入口（plugin/）**：コマンド、既定値、ロードガード
* **公開 API（autoload/${plugin_name}.vim）**：app と ui を組み合わせる唯一の場所
* **domain**：純粋なデータと規則。エディタの状態・UI・I/O・時刻・外部環境に依存しない（組み込みの純粋関数は使ってよい）
* **app**：ユースケース。domain を組み合わせ、保存と外部連携を調整する
* **storage**：永続化（読み・検証・原子的な書き）
* **ui**：バッファ、ウィンドウ、キー、プロンプト
* **deps（adapter）**：プラグインの外（外部コマンド、HTTP、ブラウザ、他プラグイン）
* **compat**：Vim/Neovim 差分（job、通知、時刻）
* **doc/**：`:help` 用ドキュメント
* **test/**：Themis テスト

`core.vim` や `util.vim` のような「何でも置ける」ファイルは作らない（肥大化の受け皿になる）。

---------

### 推奨レイヤ構造

```
plugin/${plugin_name}.vim
  └─ コマンド定義 / 既定値 / g:loaded ガード（autoload を起こさない）

autoload/${plugin_name}.vim
  └─ 公開 API（${plugin_name}#...）。コマンド引数の解析、app と ui の組み合わせ、リスナー登録

autoload/${plugin_name}/domain/*.vim
  └─ エンティティの規則・検証・純粋計算（text の無害化、スコア計算、比較関数 など）

autoload/${plugin_name}/app/*.vim
  ├─ store.vim      メモリのコピー、変更の唯一の入口 mutate()、採番
  ├─ <usecase>.vim  登録・変更・削除・起動などのユースケース
  └─ listing.vim    表示用の行の生成（UI 非依存）

autoload/${plugin_name}/storage/*.vim
  └─ 読み・検証・整形・原子的な書き（json.vim など）

autoload/${plugin_name}/ui/*.vim
  ├─ <view>.vim     専用バッファの生成・描画・キー
  ├─ prompt.vim     input / inputlist / confirm と tty 判定、注入点
  └─ picker.vim     外部 UI（fzf 等）の候補生成と選択後の遷移（任意。起動は deps 経由）

autoload/${plugin_name}/deps/*.vim
  ├─ cmd.vim        外部コマンド実行（argv List、スタブ注入点）
  ├─ http.vim       HTTP 取得（curl の契約）
  └─ <other>.vim    ブラウザ起動、他プラグイン呼び出し（fzf の起動など）

autoload/${plugin_name}/compat/*.vim
  ├─ job.vim        job_start / jobstart の差分吸収
  ├─ notify.vim     通知
  └─ clock.vim      現在時刻（テストで注入可能）
```

#### 依存の向き

表に無い組み合わせは禁止。完了時に `grep -rn '${plugin_name}#' autoload` で機械的に確認する（`call()` や `execute` で動的に組んだ関数名は grep に出ないので、動的呼び出しは公開 API 層と deps に限り、レビューで目視する）。

| 呼び出し側 | 呼んでよい相手 |
|---|---|
| `plugin/` | `${plugin_name}#…`（公開 API）のみ |
| 公開 API | `app/*`, `ui/*`, `compat/*` |
| `ui/*` | `app/*`, `ui/prompt`, `compat/*`（`ui/picker` だけ外部 UI の起動に `deps/*` を呼べる） |
| `app/*` | `domain/*`, `app/store`, `deps/*`, `compat/clock`、他の `app/*`（循環しない向きで） |
| `app/store` | `storage/*`, `domain/*` |
| `domain/*` | 他の `domain/*` のみ |
| `storage/*` | `domain/*`（読み込み時の検証に使う） |
| `deps/*` | `compat/*`, `deps/cmd`, `domain/*`（値の検証に使う） |
| `compat/*` | 無し |

* app は ui を呼ばない。非同期完了を UI に伝えたいときは、app がリスナー登録（`add_listener(Fn)`）を提供し、公開 API 層が ui の関数を登録する
* 同じ定数（マーカーの正規表現、エスケープ対象文字、区切り）を複数のモジュールに書かない。生成側のモジュールが `marker()` / `marker_pattern()` のような関数で公開し、解析側はそれを呼ぶ
* 1 関数 80 行を超えたらレビューの対象にする。分けるなら文脈を 1 つの Dictionary（`ctx`）で回す
* `autoload/` 以下の内部モジュールは言語上は外から呼べる（名前とファイルが分かれば呼べる）。公開 API 以外は「内部インターフェース。互換性を保証しない」と help に書く

#### domain 層の責務

* エンティティは Dictionary、操作は関数。**入力を変更せず、`deepcopy` した新しい Dictionary を Result に載せて返す**
* `g:`、`localtime()`、バッファ、ファイル、外部コマンドを使わない。時刻は引数 `now` で受け取る
* 識別に使う短い文字列（表示名、フォルダ名、タグ）の無害化（`text#sanitize`）は domain の入口で行う：制御文字は空白に置換（削除すると `a\tb` が `ab` になる）、不可視文字（ゼロ幅・双方向制御・ソフトハイフン・タグ文字）は除去、ただし ZWJ と異体字セレクタは残す。1 文字に続くゼロ幅文字は数個までに制限し、ゼロ幅だけの名前は空とみなす。空白の圧縮、長さ上限。**UTF-8 の妥当性**は `str2list(s, 1)` → `list2str(xs, 1)` の往復でバイト列が一致するか、サロゲートや U+10FFFF 超が無いかで検査する（第 2 引数の `1` で `'encoding'` に依存せず UTF-8 として扱う。`:help str2list()`。`str2list()` 自体は検証関数ではない）。これらをコードポイントの List 上で行うと両エディタで結果が一致し、冪等になる（正規表現・`trim()`・`substitute()` は結合文字の扱いがエディタで違う）。長さ上限は**コードポイント数**で数える（バイト数・表示幅ではない。単位を仕様に書く）。NFC/NFD の差、絵文字の ZWJ 列、異体字セレクタ、不正バイト列の入力と期待結果をテストベクトルとして両エディタで固定する。長文の本文を持つプラグインではフィールドごとに別の規則を決める

#### app 層の責務

* ユースケース 1 つ = 関数 1 つ。domain の操作を組み合わせ、`store#mutate` で保存し、deps を呼ぶ
* 外部システムを叩いてから記録する操作（起動して成功なら実績を記録）は、外部呼び出しを `mutate` の外に置く（起動と保存は 1 つの原子操作にできない。順序は「起動 → 成功なら記録」）
* 非同期結果の適用は「要求番号（seq）が最新か」「対象がまだ存在するか」「対象の鍵（URL 等）が同じか」を `mutate` の中で確認してから行う

#### storage 層の責務

* 読み・構造検証・整形・一時ファイル経由の原子的な書き。業務ルールは domain の検証関数を借りる
* 読めないファイルを「新規」と混同しない。拒否したファイルには書かない

#### ui 層の責務

* バッファとウィンドウの操作、キー、プロンプト、描画。保存形式と外部コマンドを知らない
* 専用バッファのキー操作から呼んだ app の Result は ui が通知する（その操作の入口が ui だから）。行から対象を得るときは表示文字列を解析せず、行→ID の対応表を使う

#### deps（adapter）層の責務

* **外部コマンド**：`executable()` で可用性判定、argv List、スタブ注入点（注入点は `executable()` より先に見る）
* **HTTP**：許可スキーム、リダイレクト上限、タイムアウト、応答サイズ上限、URL グロブの無効化、設定ファイルの無視、通信の無効化オプション
* **他プラグイン**：存在判定、呼び出しの成否を共通の戻り値に寄せる、未導入・未ロード時の代替
* **テスト容易性**：すべて `g:${plugin_name}_test` 経由で差し替えられる

#### compat 層の責務

* job/channel（起動、出力収集、終了通知を 1 回だけ）
* 通知（`echohl` + `echomsg`。`nvim_notify()` は非推奨なので使わない）
* 時刻（`clock#now()`。テストで注入可能）

> 目安：compat は「エディタ差分」、deps は「プラグイン外の世界」、storage は「ファイル」。domain は何にも依存しない。

---------

## 設計の共通規約

### Result（戻り値の形）

**失敗し得る関数（domain の検証・操作、app、storage、deps、compat/job）は外に例外を出さず**、次の Dictionary を返す（内部で `throw` してもよいが、同じ関数か `mutate` の `try` で受けて Result に変える）。UI はこれを見て通知する。

| キー | 型 | 意味 |
|---|---|---|
| `ok` | Bool | 成功なら `v:true` |
| `code` | String | 失敗の種別（機械判定用の snake_case。`ok` のとき `''`） |
| `error` | String | 利用者向けの文（`ok` のとき `''`） |
| `changed` | Bool | 状態が変わったか。変更操作の Result だけが明示的に付ける |
| その他 | — | 成功時の付加情報（`id`, `data`, `warnings` など。関数ごとに決める） |

```vim
" changed は既定を持たせない: 変更操作だけが明示的に付ける
function! s:ok(extra) abort
  " 予約キー (ok / code / error) は extra で上書きさせない
  return extend(copy(a:extra), {'ok': v:true, 'code': '', 'error': ''}, 'force')
endfunction

function! s:err(code, msg) abort
  return {'ok': v:false, 'code': a:code, 'error': a:msg, 'changed': v:false}
endfunction
```

* エラーコードは**設計書の 1 つの表**に全部書く。実装中に増えたら表に足す
* 失敗し得る組み込み（`json_decode`, `json_encode`, `readfile`, `writefile`, `mkdir`, `rename`, `system`, `job_start`, `jobstart`, 他プラグインの関数）は必ず `try` で包んで Result に写す。`mkdir(…, 'p')` は失敗で例外を投げる（実測 E739）が、help は「失敗・部分失敗で偽を返す」とも書くので戻り値も見る（`:help mkdir()`）
* 失敗は例外だけでなく**戻り値**でも来る：`writefile()` は -1、`rename()` は非 0、`setfperm()` は **0**（非 0 が成功）、Vim の `job_start()` は `job_status()` が `'fail'`、Neovim の `jobstart()` は 0（引数不正）/ -1（起動不能）、`system()` は `v:shell_error`。関数ごとに「例外・戻り値・`v:shell_error`・コールバック」のどれで失敗が分かるかを決めて判定する
* 失敗しない純粋計算や単純な参照（`clock#now()`, スコア計算, `current()`）は値をそのまま返してよい
* `changed` が必須の API（`mutate`）は、欠けていれば `internal_error` として拒否する
* 利用者向けの文は英語で統一し、`[${plugin_name}] ` の接頭辞を付ける

### データは Dictionary、操作はコピーを返す関数

```vim
function! ${plugin_name}#domain#item#rename(item, name) abort
  let l:clean = ${plugin_name}#domain#text#sanitize(a:name)
  if !l:clean.ok
    return l:clean
  endif
  let l:next = deepcopy(a:item)
  let l:next.display_name = l:clean.value
  let l:next.name_source = 'manual'
  return s:ok({'item': l:next, 'changed': v:true})
endfunction
```

* 「状態を変えるロジックを、その状態の規則を知っている場所（domain）に集める」のが目的。他の層が `let item.display_name = …` と直接書き換えない
* 無変更で終わる操作は `ok: v:true, changed: v:false`

### 変更の唯一の入口（`app/store#mutate`）

永続化する状態の変更は、**すべて 1 つの関数を通す**。

```
mutate(Fn):
  1. ファイルを読む（失敗したら何も書かずに返す）   ← 壊れたファイルを守る唯一の砦
  2. Fn(deepcopy(data)) を try で呼ぶ                ← 例外は internal_error
     戻り値が Dictionary でない / ok なのに data が無い → internal_error
  3. Fn の Result が !ok → そのまま返す
  4. changed: false → 保存しない。Fn が返した data は捨て、1 で読んだディスク状態をメモリに反映して付加情報だけ返す
  5. 保存（読んだときの mtime を渡す楽観ロック）
  6. conflict なら 1 からやり直す（1 回だけ）         ← Fn は純粋な変換なので再適用できる
  7. メモリを更新し、Fn の付加情報を載せて返す
```

上は簡易方式（下の「競合検出」）。ロック方式では 1 の前に保存先ディレクトリを確保してから（`mkdir(dir, 'p', 0700)`。無いと `mkdir(lockdir)` が親ディレクトリ不在で失敗する）ロックを取り、7 の後（`finally`）で解放する。ロック保持中は競合が起きないので 6 の再試行は不要で、取得失敗が `conflict` になる。

* `Fn` は「現在のデータを受け取り、変換後のデータを `data` に載せた Result を返す」純粋な関数。`function('s:add_fn', [url, name, now])` のように partial で引数を束ねる
* 再試行できる条件：`Fn` が外から受け取る値（`now`、入力）は呼び出し前に固定し、ID の採番はデータの中のカウンタから行い、外部副作用（コマンド実行・通知）は `Fn` の中に入れない
* `changed: false` は「永続化対象のデータが不変」の意味。`Fn` がデータを変えたのに `changed: false` を返すと画面とディスクがずれるので、`mutate` は `changed: false` のとき返却データを採用しない（上の 4）。ただし 1 で読んだ最新のディスク状態はメモリに反映する（別プロセスの変更を取り込む）
* 競合検出は要件で選ぶ。**簡易方式**（`getftime()` の楽観ロック）は秒精度で、同一秒内の 2 書き込みと「確認から `rename()` までの窓」は検出できない。更新損失を許容できる単一利用者のローカルツールだけに使い、残る窓は help の Limitations に書く。複数のエディタや外部ツールからの同時更新が普通にある用途では、データに `revision` を持たせて保存時に比較し、ロックの下で読込→変換→保存を行う方式を標準にする。ロックは `mkdir(lockdir)`（`'p'` 無し）の原子性で取る（help と小さな実測で確認した手順。実プラグインでは未実証）：成功（1）で取得。例外（E739）は既存ディレクトリ以外（親が無い、権限）でも出るので、`catch` で `isdirectory(lockdir)` を見て、あれば `conflict`、無ければ `lock_failed` に分ける。`lockdir` は `resolve()` 後の保存先に対応付ける（同じファイルを別の名前で開いても同じロックになる）。所有者の pid を中のファイル（`lockdir/pid`）に書き、期限切れ（`getftime(lockdir)` が一定時間より古い）の判断基準と手動解除の手順を help に書く。解放は**取得に成功したときだけ** `finally` で行い、`delete(lockdir . '/pid')` が `0` を返してから `delete(lockdir, 'd')`（`'d'` は空でないディレクトリを消せず -1 を返す。`'rf'` は使わない）。解放に失敗したら警告してロックが残ったことを隠さない。ロック保持中は mtime の比較は不要（`save` の `expected_mtime` を `v:null` にして飛ばす）。ロックで守れるのは同じロックを使う書き手だけで、外部ツールの直接編集は守れないと help に書く。ネットワークファイルシステムでは `mkdir` の原子性を前提にできないので保証外と書く。どちらを採ったかを設計書に書く
* 読み取り（`current()`）はメモリのコピーを返す。表示はメモリから、変更は毎回ディスクから

---------

## 実装パターン

### storage（JSON ファイル）

* **読み（`load(path)`）**：`getftype(path) ==# ''` のときだけ「新規 = 空ストア」。**存在するが読めない**（権限、ディレクトリ、死んだマウント）は `load_failed`。`!filereadable()` を新規扱いにすると、次の保存で既存ファイルを置き換えて消す
* `getftype(path)` が `file` / `link` 以外（FIFO、ソケット、デバイス）なら読まずに `load_failed`（`readfile()` が無期限にブロックする）。シンボリックリンクは `resolve(path)` した先についても同じ種類検査を行い（リンク先が FIFO やディレクトリなら拒否、リンク切れは `load_failed`）、一時ファイルと `rename()` の対象はリンク先にする（そうしないとリンクが通常ファイルに置き換わる。dotfiles へリンクする利用者がいる）。検査から書き込みまでにリンク先が差し替えられる窓（TOCTOU）は Vimscript では塞げないので、help の Limitations に書く
* 読む前に `getfsize(path)` を上限（例 10 MiB）と比べ、`size < 0 || size > max` なら `readfile()` せず `load_failed`（検査と読込の間に膨らむファイルは防げない通常ファイル向けの事前防御。`-1` は見つからない、`-2` は Number に収まらない。`:help getfsize()`。新規ストアの判定は `getftype()` 側で済んでいる）。読んだ後も要素数・文字列長・入れ子の深さをスキーマの上限で検査する
* 構造・型・不変条件・**ID の重複や参照切れ**は `invalid_data` で拒否し、**拒否したファイルには書かない**。自動で直せる軽い不整合（一意性に影響しない重複、採番カウンタが既存 ID 以下）だけ `warnings` に積んで寛容に読む（手編集 1 行で全停止しないように）。補正は読込直後にメモリ上で行う（採番カウンタは `max(既存 ID) + 1` に上げる）。補正できない値（上限超え）は `invalid_data`
* **書き（`save(path, data, expected_mtime)`）**：`encode` → 楽観ロック（`expected_mtime` が `v:null` でなければ、読んだときの存在状態と今の存在状態・mtime を比べる：読んだとき無かった（`expected_mtime == -1`）のに今ある、読んだとき在ったのに今無い（別プロセスが消した）、在って mtime が違う、在るのに読めない、のいずれも `conflict`。ロック方式は `v:null` で飛ばす）→ `try` の中で `mkdir(dir, 'p', 0700)`（新規作成は 0700 で。`resolve()` 後の実際の保存先ディレクトリが**他人に書ける**権限なら、下の前提が崩れるので `unsafe_dir` で保存を拒否する。他人に読めるだけなら 1 回警告して続ける）→ `delete(tmp)` → `writefile([], tmp)` で空ファイルを作る（戻り値 `-1` と例外を失敗とみなす）→ `setfperm(tmp, 'rw-------')`（**非 0 が成功、0 が失敗**。`:help setfperm()`。失敗なら内容を書かない）→ `writefile(lines, tmp)`（同じく `-1` と例外を失敗とみなす。既存ファイルへの `writefile()` は権限を保つので、内容が他人に読める瞬間を作らない）→ `rename(tmp, path)`（`0` が成功）。失敗したら `finally` で自分が作った `tmp` を消す。Vimscript には排他的作成（`O_EXCL`）が無いので、この手順は**データディレクトリが利用者専用（0700）であること**を前提にする。他人が書ける共有ディレクトリでの安全性は主張せず、Limitations に書く。`tmp = path . '.tmp.' . getpid() . '.' . localtime() . '.' . s:seq`（保存ごとに増やす連番。同じ名前を使い回さない）。権限の検査は保存のたびに行う（利用者が後から変えることがある）
* `rename()` は同一ファイルシステム内で原子的に置き換えるが、電源断に対する永続性や Windows での原子性は保証されない。永続性を上げたいなら `writefile(lines, tmp, 's')` で `fsync()` させる（既定は `'fsync'` オプションに従う。`:help writefile()`）。復旧時にどれを信用するか（元ファイル → `.bak` → 残骸の順。残骸は信用しない）と、保証できない環境を help の Limitations に書く
* `writefile` に `'b'` を付けない（末尾改行が無くなり、git diff に毎回 `\ No newline` が出る）
* 整形は自前で書く（`json_encode()` は 1 行で diff が読めない）。キー順固定、1 要素 1 行。文字列は要素ごとに `json_encode()` でエスケープし、`try` で包む。Float は `printf('%.4f')` だと `0.00004` が `0` に、`1e300` が 300 桁になる。往復して同じ型（Float は Float）に戻る表現を選び、JSON で表せない NaN / Inf は保存前に `invalid_data` で拒否する。負値や範囲はフィールドごとのスキーマで決める（負値が無意味なフィールドだけ拒否する）。極小・極大・整数の往復をテストで保証する。手編集や git 管理を想定しないデータなら `json_encode()` の 1 行で済ませてよい
* Dictionary の比較では `1 == 1.0` が偽。読み込み時に Float を正規化（`x * 1.0`）し、整数フィールドは Number を要求する
* `schema_version` を持ち、上位は拒否、下位は読み込み時に**メモリ上だけ**移行する（`load` はディスクに書かない）。移行後の形式をディスクに書くのは次の保存のときで、その保存の中で元ファイルを `.bak` にコピーしてから `rename()` する（`.bak` も一時ファイルと同じ手順（空作成 → 600 → 内容 → `rename`）で作り、`.bak` は上書きしない。既にあれば連番。コピーに失敗したら保存を中止して `save_failed`。ロック方式ならロック保持中に行う。旧スキーマのファイルを置いて移行 → 保存 → `.bak` → 旧版で `.bak` を読む、を通しでテストする）。壊れたファイルは上書きせず、退避先と復旧手順を help に書く
* 一時ファイルは作った保存試行が所有し、成功・失敗にかかわらず自分で消す（`finally`）。**他のプロセスの残骸は自動で消さない**（pid と経過時間では使用中かどうか判定できない。消すと相手の保存が失敗する）。残骸を掃除する操作を設けるなら明示的な保守コマンドにし、`readdir()` で名前のプレフィックス（`<ファイル名>.tmp.`）を完全一致させる（`glob()` はパス中の `[` `*` をパターンとして解釈する）
* 表示中に別プロセスがファイルを変えても自動では追わない（監視しない）。再読込コマンドを用意し、未保存の編集があるときの扱い（破棄の確認）を決める

### compat/job（非同期）

契約：`start(argv, OnExit, opts)` → Result（起動失敗は Result で返し、コールバックは呼ばれない）。`OnExit({exit_code, stdout, reason})` を**ちょうど 1 回**呼ぶ。`reason` は `exited | output_too_large | stopped | timeout`。`reason` が `exited` 以外のとき、呼び出し側は `stdout` を正常な結果として適用しない（切り詰められている）。stderr は捨てる（累積しない）。`opts.timeout_ms` を持ち、超えたら停止して `timeout` で 1 回だけ呼ぶ。停止・上限超過・タイムアウトも同じ完了管理を通す（下の「両方待つ」と、読み取りを自分で閉じた場合の例外）。上限超過のテストは実ジョブ（無限に出力するコマンド）で両エディタに持つ。保証範囲は「起動に成功してからエディタが動いている間」。

* 共通：`executable(argv[0])` を先に見る（Neovim の `jobstart()` は不存在コマンドで E475 を画面に出す）
* **Vim**：`job_start(argv, {out_mode: 'raw', out_cb, err_io: 'null', in_io: 'null', close_cb, exit_cb})`。`exit_cb` は最後の `out_cb` より先に来ることがある。**`close_cb` と `exit_cb` の両方が来た時点で 1 回だけ** OnExit を呼ぶ。Unix では不存在コマンドでも `job_status` は `'fail'` にならず `'dead'`（exit 122）で通常経路を通る
* **Neovim**：`jobstart(argv, {on_stdout, on_stderr, on_exit, stdin: 'null'})`。`on_stdout` の `data` は NL で分割された List で、前回の最後の要素に続く。各コールバックの `join(data, "\n")` を `''` で連結すれば**テキスト出力**は復元できる（help の「前回末尾の要素に今回先頭を連結」と等価）。`['']` は EOF（`:help channel-callback`）。実測では EOF → `on_exit` の順に来るが help は順序を約束していないので、通常終了は Vim と同じく **stdout の EOF と `on_exit` の両方が来た時点で 1 回だけ** OnExit を呼ぶ。ただし自分で `chanclose(job, 'stdout')` して止めた後は EOF の `['']` は来ない（実測）。読み取りを自分で閉じた（`stdout_closed`）場合は EOF を待たず `on_exit` だけで完了させる。Vim の `job_stop()` 後は `close_cb` が来る
* **停止**：`job_stop(job)` / `jobstop(id)` の戻り値 1 は「シグナルを送れた」であって終了ではない（`:help job_stop()`）。TERM を送って猶予（例 2 秒）を待ち、まだ生きていれば `job_stop(job, 'kill')` / Neovim は `jobstop()` が KILL まで面倒を見る。`reason` は停止を決めた時点で確定し、その後に来る終了イベントで OnExit を 1 回だけ呼ぶ。KILL 後も終了イベントが来ない場合はタイマーで打ち切り、`stopped` で呼ぶ
* **出力の上限**（`opts.max_bytes`）：バイト数を累積し、超えたら Vim は `job_stop()`、Neovim は `chanclose(job, 'stdout')` で読み取りを止めてから `jobstop()`。Neovim は 1 回の `on_stdout` が何 MiB も運ぶことがあるので **`join` する前に必要な行数だけ採る**（main loop が詰まっている間に Neovim 側で巨大な List ができるのは Vimscript では防げない。実経路では上流で絞る）。`stdout_buffered` は EOF まで呼ばれないので上限と両立しない
* ctx は partial（`function('s:on_exit', [ctx])`）で渡す。ただし **Vim の Job オブジェクトは参照を保つ**（help の要求。`s:jobs` の規約は実プラグインでは未実証）：`:help job_start()` は「参照が無くなると Job は削除され、stdin / stderr が閉じられる」と書いている。起動に成功したら `ctx.job` に入れ、さらに `s:jobs[id] = ctx` で OnExit まで保持し、OnExit を呼んだ後に消す（登録表を持たない規約の唯一の例外。`_reset()` と Vim 終了時の扱いもここで決める）。Neovim の job id は Number なので同じ表に入れてよい

### deps/cmd（外部コマンド）

* argv は常に **List**。文字列連結も `shellescape` 連結も原則しない
* `system(List)` は **Vim 9.2.0250 で入った**（`:help E1575`。それ以前は E730）。判定は `has('nvim') || has('patch-9.2.250')`。派生版向けに試行判定を足すなら `try | call systemlist(['printf', '']) | catch` を補助に使う（例外の有無で List の受理だけを見る。`printf` が無い環境では非対応側に倒れる）。非対応ならそのときだけ `shellescape` 連結に落とす（シェル経由になり実行の意味が変わるので、特殊文字を含む引数の統合テストを両エディタで持つ。`shellescape()` は `&shell` に依存し、Windows では同値にならない）。もう 1 つの代替は `job_start(List)` を同期で待つ方式（シェルを通さない。Vim 8.0 以降で使える）。Neovim は昔から List 可
* `system()` の出力には stderr が混ざる（String / List とも。両エディタで実測）。stderr を分けたいなら job を使う。契約に書く
* **スタブ注入点は `executable()` より先に見る**（実バイナリが無い CI でもスタブが効く）
* 「起動だけして戻りたい」コマンド（ブラウザ起動）は、Vim の `system(List)` だと stdout を握った孫プロセスの終了まで待つ。Vim では `job_start(argv, {in_io/out_io/err_io: 'null', stoponexit: ''})` で起動し、本体の終了だけを数秒待つ
* 利用者入力が `system` / `execute` / `normal!` / `printf` の書式に届く経路を作らない。`execute` の対象は固定文字列だけ

### deps/http（curl）

* argv の例：`['curl', '-q', '-g', '-fsL', '--compressed', '--proto', '=http,https', '--proto-redir', '=http,https', '--max-redirs', '5', '--connect-timeout', '5', '--max-time', '10', '--max-filesize', '1048576', '-A', '<UA>', '--', url]`
* `-q`：`~/.curlrc` を読まない。`-g`：URL のグロブを無効化。`--proto-redir`：リダイレクト先を http(s) に限定（`-L` の既定は FTP も許す）。`--`：URL が `-` で始まる事故の二重防御
* `--max-filesize` は curl 8.4.0 より前では事前にサイズが分からない転送に効かない。`--compressed` と併用したとき展開後のサイズに効くのは curl 8.20.0 以降（curl の資料。手元では未確認）。job 側のバイト上限を必須の防御にする
* `--proto` はスキームの制限であり、ローカルホストやプライベートアドレスへの到達（SSRF）は防がない。利用者が自分の URL を登録するローカルツールでは受容して Privacy / Limitations に書く。他者の入力を受ける用途なら接続先の検証を deps/http に足す
* タイムアウトは curl に任せる。リトライはしない（利用者の明示操作で再試行）
* 応答の解析（`<title>` 抽出など）は入力長に対して線形なアルゴリズムで書き、閉じていないタグ × 10,000 や 1 MiB の整形式で時間を測る。コールバックから同期で呼ぶので、超線形だとエディタが固まる

### compat/notify, compat/clock

* 通知は `echohl` + `echomsg`（`:messages` に残る。`echohl None` への復帰は `try … finally` で）。通知の所有者は**その操作の入口**：コマンド・公開関数から入った操作は公開 API 層が、専用バッファのキーから入った操作は ui が通知する。app / storage / deps / compat は Result を返すだけで通知しない。同じ Result を 2 か所で通知しない。`nvim_notify()` は Neovim 0.11 以降**非推奨**（`deprecated.txt`）なので使わない。`warn_once(key, msg)` を用意し、不正な `g:` 値の警告は 1 回だけ
* `clock#now()` は `g:${plugin_name}_test.now` があればそれ、無ければ `localtime()`。app と ui はこれを使い、domain には引数で渡す

### ui/prompt

* `has_tty()`（対話できるかのヒューリスティック）：Neovim は `!empty(nvim_list_uis())`。Vim は `has('gui_running') || (has('ttyin') && mode(1) !~# '^c[ve]')`。**偽のときは `input()` / `inputlist()` / `confirm()` を呼ばず `no_tty` を返す**（tty 無しの `input()` は Vim `-es` でスクリプトを打ち切り、Neovim headless で無期限ブロックする。端末から起動したヘッドレス Neovim は `has('ttyin')` が 1 になるので、`nvim_list_uis()` で見る）。リモート UI や組み込み環境では外れ得るので、呼び出し側が非対話を明示できる注入点も残す
* `input()` の `<Esc>` は空文字を返すだけで例外は出ない。取消は `Vim:Interrupt`（CTRL-C）。Neovim では `input({'cancelreturn': "\<Esc>"})` で `<Esc>` も取消にできる。利用者の `cnoremap`（例 `cnoremap <C-c> <DEL>`）は `input()` にも効くので、「空で確定 = 取消」の経路も残す
* プロンプトは**注入可能にする**（`g:${plugin_name}_test.prompt = {text, choose, confirm}`）。ヘッドレスのテストではプロンプトを通る経路を直接テストできないため、これが無いとキー操作の大半がテスト不能になる
* `inputlist()` の 0 は「選ばなかった」。候補は 1 から始める

### ui/<view>（専用バッファ）

前半は専用バッファを持つプラグイン全般の基本。後半の「編集可能モード」は `:w` で差分を適用する UI 向けの応用レシピで、必要なときだけ採る。

* バッファは `bufadd('${plugin_name}://${view}')`（無ければ `bufnr(name, 1)`）→ `setbufvar` で `buftype=acwrite bufhidden=hide noswapfile nobuflisted nomodifiable undolevels=-1 nomodeline` → `silent call bufload(buf)`（Vim は silent にしないと `[New DIRECTORY]` が出る）。オプションはロード前に付ける（`setbufvar` は未ロードのバッファにも効く）。`bufload()` は未ロードのバッファで `BufReadCmd` / `BufEnter` / `BufWinEnter` を起こし得る（両エディタで実測。ロード済みなら何もしない）ので、利用者の autocmd が走ることを前提にする。`acwrite` にすると `:w` を `BufWriteCmd <buffer>` で受けられる。編集可能モードを持たないなら `nofile`
* 初期化順（窓に表示した後、そのバッファで）：`setlocal filetype=${plugin_name}`（`setbufvar` ではなく `setlocal` にすると FileType autocmd が確実に走り、利用者の `autocmd FileType ${plugin_name} nmap <buffer>` が先に入る）→ `<Plug>(${plugin_name}-<action>)` を定義 → `hasmapto()` を見て既定キー → ハイライトグループ（`hi default link`、`ColorScheme` で再実行）
* **既定キーはバッファローカル `<nowait>`**（グローバルの接頭辞マップに勝つ）。利用者の接頭辞マップを尊重したいなら、`mapcheck(key)` が非空のキーだけ `<Leader>{key}` に逃がす選択肢もある。どちらでも、**開く・閉じる・ヘルプのキーは常に素のキー**にする（`mapcheck` は `qr` があるだけで `q` を衝突と見るため、閉じるキーまで逃げるとマクロ記録が始まる）。ヘルプには実際に張ったキーを表示する。既定キーを一括で無効化する `g:${plugin_name}_no_default_mappings` を用意し、`<Plug>` だけで組めるようにする
* **行→項目の対応表**（`b:${plugin_name}.rows`、添字 = 行番号 − 1）を持ち、表示文字列を解析しない。ハイライトは `syntax match` ではなく rows の種別から `matchaddpos()` で付ける（表示名に区切り文字が入っていても誤反応しない）。`matchaddpos` はウィンドウ単位なので id を `w:` に保持して `matchdelete`。`clearmatches()` は他プラグインのマッチも消す
* **再描画は対象バッファ番号を明示する関数で行う**（help と小さな実測で確認した手順。実プラグインでは未実証）：既存行への `setbufline(buf, …)` 上書き + `deletebufline(buf, …)`。非同期コールバックやタイマーから呼ばれるときはカレントバッファが利用者の編集中のファイルであり得るので、`setline()` / `setlocal` のようにカレントに作用する関数を描画に使わない（使うと別ファイルを書き換え、`nomodified` で未保存の印まで消す）。描画前に `bufexists(buf)` と `getbufvar(buf, '&filetype')` が自分のものかを再確認する。バッファは `nomodifiable` なので、描画中だけ `setbufvar(buf, '&modifiable', 1)` にし、`try … finally` で `0` に戻す（`setbufline()` は `modified` を立てるので、閲覧専用バッファでは最後に `setbufvar(buf, '&modified', 0)`。両エディタで実測）。`%delete _` は fold の開閉状態を消す（上書きなら残る）。rows を更新してから `setline` する（foldexpr が rows を参照する）。編集可能モードのバッファが `modified` のときは自動再描画しない（未保存の編集を消す）。通知して利用者に `:w` か `:e!` を選ばせる
* 描画はバッファ部分（1 回）とウィンドウ部分（`win_findbuf()` の各窓で `win_execute()`：カーソルを項目 id で復元、`leftcol` だけ `winrestview`、matches 付け替え）に分ける。`win_gotoid` は別タブへの切替を起こすので使わない
* ウィンドウローカルなオプション（`foldmethod`, `conceallevel`, `wrap`…）は `setlocal` では現在窓にしか効かない。全窓に `win_execute` し、`BufWinEnter <buffer>` と `WinEnter <buffer>` でも適用する（同じバッファの `:split` は `BufWinEnter` を起こさない）。`BufWinLeave` は他の窓に表示中だと発火しないので、窓のマッチの掃除は `BufLeave` + `timer_start(0)` で見直す
* 折りたたみは独自実装せず `foldmethod=expr` + `foldexpr` が rows を読む。`za` `zM` `zR` が無料で付く
* **非同期完了時の再描画は遅延判定を通す**：`getcmdtype() !=# ''`（プロンプト中）や挿入・ビジュアル・オペレータ待ちなら `timer_start()` で後回し。`input()` の最中にも job コールバックは走る。`state()` はタイマー・ジョブのコールバック内で常に `c` を含むので、`state('mwoScx')` を条件にすると永久に実行されない
* 編集可能モード（`:w` で差分適用）を持つなら：行末に conceal した ID マーカー（`matchadd('Conceal', pat, 10, -1, {'conceal': ''})`）、`BufWriteCmd` で解析 → 差分 → 1 回の `mutate`、`:e!`（`BufReadCmd`）で破棄、`modified` が落ちないと `:wq` は閉じない（= 失敗時は閉じない）。help は `BufWriteCmd` の中で**バッファ内容を変えない**ことを求めているので、成功後の再描画はハンドラ内で直接行わず `timer_start(0, …)` で後に回す。文法は「URL トークンを先に探す → 末尾 `/` ならフォルダ → それ以外はエラー」のように**優先順位を明記**し、親の決定は木の整合性（直上で最初に浅い行が、ちょうど 1 段浅い親であること）で検査する

### plugin/ と公開 API

* `g:` の値の検証は使う側で行い、不正ならハードコードの既定に戻して `warn_once`。`g:` が壊れていても動く
* 公開 API（`${plugin_name}#open()` など）は Result を返し、コマンド・公開関数から入った操作の失敗はここで 1 回通知する。`command!` は公開 API を呼ぶだけでよい（Result は捨ててよい。通知済み）。関数を直接呼ぶ利用者は Result で判定できる。利用者のマッピングは**コマンドに向けさせる**（遅延ロードされると関数は E117）
* リスナー登録は冪等にする：`add_listener` 側で同じ Funcref の重複登録を無視し（`index(s:listeners, Fn) < 0` のときだけ `add`）、公開 API 層のフラグは補助にする。テスト用の `_reset()` がリスナーを消すなら、公開 API 側のフラグも同時に戻す。`open()` のたびに登録すると 1 回の変更で何度も再描画される
* `-bar` は引数に `|` が入り得るコマンドには付けない

---------

## 実装手順

0. 雛形を作る

* LICENSE / README.md / Makefile / `.gitignore` / `.mcp.json`
* `plugin/`, `autoload/${plugin_name}/{domain,app,storage,ui,deps,compat}/`, `doc/`, `test/`

1. 要件定義（ユーザー + orchestrator）
   目的 / スコープ外 / 未決事項を列挙して全部決める。決定ごとに理由と代償を 1 行。

2. 基本設計（orchestrator）
   ユースケース（主要フロー・代替フロー・事後条件）、概念モデル（Mermaid の ER 図）、処理フロー（シーケンス図）、非機能、設計判断と代替案。コードは書かない。

3. 詳細設計（orchestrator）
   層と依存の表、Result とエラーコードの表、各層の関数とデータ形、保存形式、非同期と外部連携の契約、UI の描画規則、テスト設計（ファイルごとに保証すること）、要件からの変更点の表。「実装者が迷わない」が基準。

4. 設計レビュー（orchestrator）
   観点を分けた独立したレビュー（要件整合 / 妥当性 / データと非同期の**事実確認**（help と実測）/ UI・セキュリティ・実装可能性）。規模に応じて並列に。指摘を Blocker / High / Medium / Low に分け、Medium 以上は設計書に反映して版を上げる。

5. 計画承認（ユーザー）
   承認後にのみ実装へ進む。

6. 実装（Task エージェント）
   雛形 → **最もリスクの高い技術境界を先に動かして両エディタで検証**（非同期を使うなら `compat/job` と待ち方、外部コマンドなら `deps/cmd`）→ domain → storage → app → deps / compat → ui → 公開 API → 応用機能 → e2e → doc / README。各段で両エディタのテストが通ってから次へ。

7. テスト（Task エージェント）
   `make test-vim` / `make test-nvim` / `make lint`。pty で実際にキーを打つ E2E（`-u NONE` と利用者の実 vimrc の両方。終了コード、バッファ内容、データファイルの内容、`:messages` に `E\d+` が無いことを組み合わせて判定する。`E\d+` の不在だけでは合格にしない）。

8. 敵対的レビュー（Task エージェント）
   観点を分けたレビュー（設計との差分 / 正しさ・セキュリティ・データ保全を実行して攻撃 / pty モンキーテスト）。修正したら、修正範囲と、その変更が触った契約（保存形式・非同期・公開 API）の依存先をもう 1 回レビューする。

9. ドキュメント更新（Task エージェント）
   README / `doc/${plugin_name}.txt` を実装と一致させる（Privacy と Limitations を含む）。

10. 提出物（Task → ユーザー）

   * 差分（変更点が追える形）
   * 互換性メモ（分岐点・フォールバック）
   * Vim/Neovim の簡単なテスト手順
   * チェックリスト（確認観点）の確認結果
   * 要件定義の末尾に「実装結果と逸脱」（実装先、規模、テスト件数、性能値、設計からの変更と理由、見つけた設計の誤り、既知の制限、利用者がやること）

### レビューで毎回見る観点

* 依存の向き（`grep -rn '${plugin_name}#' autoload`）。グローバル autocmd の有無。`plugin/` が autoload を起こしていないか
* Result 契約を破る `throw` や、`try` の無い失敗し得る組み込み
* 読めないファイル・壊れたファイル・権限・ディレクトリ・一時ファイルの残骸で**既存データが消えないか**（実際に権限を落として試す）
* 悪意ある入力で**固まらないか**（1 MiB の整形式、閉じていないタグ × 10,000、無限に出力するコマンド、不正 UTF-8、制御文字、RTL 制御、2049 文字の URL）
* 利用者入力が `system` / `execute` / `normal!` / 書式文字列に届く経路
* 利用者の実 vimrc でキーが潰れないか（接頭辞マップ、`<Leader>`、`;`⇄`:` 入替、`cnoremap`）
* 両エディタの差（`json_encode` の UTF-8、`system(List)`、`jobstart` のエラー表示、`input` の取消、`has('ttyin')`、結合文字）
* 1,000 件での描画・保存・編集の時間

---------

## 修正手順

1. 修正要件入力（ユーザー）：期待動作 / 実際 / 再現手順（可能なら）
2. 理解・質問（orchestrator）：不明点を潰す
3. 修正計画作成（orchestrator）：`./tmp/fix-plan-...md`（原因仮説・影響範囲・追加する回帰テスト）
4. 計画承認（ユーザー）
5. 修正実装（Task）：先に回帰テスト追加 → 修正 → lint/test 成功（両エディタ）
6. ドキュメント更新（Task）
7. 確認観点チェック（Task）：性能/安全性/データ保全/互換性のチェックリストを確認して提出

---------

## 配布と互換性

* 導入方法を README に 2 つ以上書く（vim-plug、ネイティブパッケージ）。ネイティブパッケージは `pack/*/start/`（起動時に自動ロード）と `pack/*/opt/`（`:packadd` で手動ロード。`:help packages`）の両方の配置例を書く。`pack/*/opt` のプラグインは `:packadd` するまで `plugin/` も読まれないので、コマンドも無い。vim-plug の `on:` は別の仕組みで、指定したコマンドだけを先に定義してロードの引き金にする（`<Plug>` と関数は読み込み後まで無い）。どちらも help に書く
* 本体の対応 OS（Unix / macOS / Windows）と、開発ツールチェーンの対応 OS（Makefile と pty E2E は Unix 前提）を分けて宣言する。Windows を本体で支えるなら、パス・権限・`rename()` の差と、Windows 用のテスト手順を書く
* このガイドで使う組み込み関数の導入版（`version8.txt` の Solution 行から）。最低版を Vim 9.0 / Neovim 0.9 にすれば `system(List)` 以外は全部使える

| 関数 | Vim | 備考 |
|---|---|---|
| `matchaddpos()` | 7.4.330 | |
| `job_start()` | 7.4.1274 | |
| `json_encode()` / `json_decode()` | 7.4.1470 | |
| `setfperm()` | 7.4.1516 | |
| `win_findbuf()` | 7.4.1558 | |
| `getbufinfo()` | 7.4.2204 | |
| `setbufline()` | 8.0.1039 | |
| `appendbufline()` / `deletebufline()` | 8.1.0037 / 8.1.0039 | |
| `readdir()` | 8.1.1120 | |
| `str2list()` / `list2str()` | 8.1.1122 | |
| `win_execute()` | 8.1.1418 | 別タブの窓への対応修正は 8.1.1832 |
| `bufadd()` / `bufload()` | 8.1.1610 | |
| `state()` | 8.1.2047 | |
| `system(List)` / `systemlist(List)` | 9.2.0250 | `has('patch-9.2.250')` で判定 |
* 版は SemVer の git タグ。保存形式を変える変更、既定キーや既定値を変える変更は破壊的変更として CHANGELOG に書き、`schema_version` の移行と `.bak` の残し方、戻し方（旧版 + `.bak`）を help に書く
* `g:${plugin_name}_test` のようなテスト用注入点は**開発者向けの内部インターフェース**と help に明記し、互換性を保証しない

---------

## 実装ファイルについて

雛形のプレースホルダは次の 3 つ。置換後に autoload の関数名とファイルパスが一致しているか（E746）を確認する。

| プレースホルダ | 例 | 用途 |
|---|---|---|
| `${plugin_name}` | `sample_plugin` | ディレクトリ名、関数の接頭辞、`g:` の接頭辞、バッファ名 |
| `${PluginName}` | `SamplePlugin` | README の見出し、テスト補助関数の接頭辞（グローバル関数は大文字始まり） |
| `${PluginCommand}` | `SamplePluginOpen` | `:command` 名 |

### 基本的なディレクトリ構成例

```
LICENSE
Makefile
README.md
doc/
  ${plugin_name}.txt
plugin/
  ${plugin_name}.vim
autoload/
  ${plugin_name}.vim
  ${plugin_name}/
    domain/*.vim
    app/*.vim
    storage/*.vim
    ui/*.vim
    deps/*.vim
    compat/*.vim
test/
  .themisrc
  domain_*.vim  app_*.vim  storage_*.vim  deps_*.vim  compat_*.vim  ui_*.vim  plugin_*.vim  e2e_*.vim
```

### `plugin/${plugin_name}.vim` の基本形（入口を軽く）

```vim
if exists('g:loaded_${plugin_name}')
  finish
endif
let g:loaded_${plugin_name} = 1

let s:save_cpo = &cpoptions
set cpoptions&vim

" default options（未設定のときだけ）
if !exists('g:${plugin_name}_open')
  let g:${plugin_name}_open = 'vsplit'
endif

" -bar は付けない（引数に | が入り得るとき）
command! -nargs=? ${PluginCommand} call ${plugin_name}#open(<q-args>)

let &cpoptions = s:save_cpo
unlet s:save_cpo
```

### `autoload/${plugin_name}.vim`（公開 API を集約）

```vim
function! ${plugin_name}#open(...) abort
  " app が持つ非同期イベントに ui を接続する唯一の場所。
  " add_listener は同じ Funcref の重複登録を無視する（冪等）ので毎回呼んでよい
  call ${plugin_name}#app#title#add_listener(function('${plugin_name}#ui#tree#on_changed'))
  let l:r = ${plugin_name}#ui#tree#open(get(a:000, 0, ''))
  if !l:r.ok
    " 失敗の通知はここで 1 回だけ。コマンドは Result を捨ててよい
    call ${plugin_name}#compat#notify#error(l:r.error)
  endif
  return l:r
endfunction
```

### `autoload/${plugin_name}/compat/notify.vim`

```vim
let s:warned = {}

function! s:emit(group, msg) abort
  if a:group !=# ''
    execute 'echohl' a:group
  endif
  try
    " 改行・制御文字は 1 行に潰す（echomsg の表示崩れと :messages の汚染を防ぐ）
    echomsg '[${plugin_name}] ' . substitute(a:msg, '[[:cntrl:]]', ' ', 'g')
  finally
    echohl None
  endtry
endfunction

function! ${plugin_name}#compat#notify#info(msg) abort
  call s:emit('', a:msg)
endfunction

function! ${plugin_name}#compat#notify#warn(msg) abort
  call s:emit('WarningMsg', a:msg)
endfunction

function! ${plugin_name}#compat#notify#error(msg) abort
  call s:emit('ErrorMsg', a:msg)
endfunction

function! ${plugin_name}#compat#notify#warn_once(key, msg) abort
  if !has_key(s:warned, a:key)
    let s:warned[a:key] = 1
    call ${plugin_name}#compat#notify#warn(a:msg)
  endif
endfunction
```

### `autoload/${plugin_name}/compat/clock.vim`

```vim
function! ${plugin_name}#compat#clock#now() abort
  let l:test = get(g:, '${plugin_name}_test', {})
  if type(l:test) == v:t_dict && type(get(l:test, 'now', v:null)) == v:t_number
    return l:test.now
  endif
  return localtime()
endfunction
```

### `autoload/${plugin_name}/deps/cmd.vim`（外部コマンド吸収の例）

```vim
" 戻り値は dict に統一（ok / code / error / exit_code / stdout）
" stdout には stderr が混ざる（両エディタ）
let s:list_argv = -1

" system(List) は Vim 9.2.0250 から（:help E1575）。Neovim は昔から可
function! s:list_argv_supported() abort
  if s:list_argv < 0
    let s:list_argv = has('nvim') || has('patch-9.2.250')
  endif
  return s:list_argv
endfunction

function! ${plugin_name}#deps#cmd#run(argv, opts) abort
  if type(a:argv) != v:t_list || empty(a:argv)
    return {'ok': v:false, 'code': 'not_executable', 'error': 'command not found: (empty argv)', 'exit_code': -1, 'stdout': ''}
  endif
  " テスト用スタブ注入点は executable() より先に見る
  let l:test = get(g:, '${plugin_name}_test', {})
  if type(l:test) == v:t_dict && type(get(l:test, 'cmd_runner', 0)) == v:t_func
    return call(l:test.cmd_runner, [a:argv, a:opts])
  endif
  if !executable(a:argv[0])
    return {'ok': v:false, 'code': 'not_executable', 'error': 'command not found: ' . a:argv[0], 'exit_code': -1, 'stdout': ''}
  endif
  try
    if s:list_argv_supported()
      let l:out = system(a:argv)
    else
      let l:out = system(join(map(copy(a:argv), 'shellescape(v:val)'), ' '))
    endif
  catch /.*/
    return {'ok': v:false, 'code': 'exit', 'error': v:exception, 'exit_code': -1, 'stdout': ''}
  endtry
  let l:code = v:shell_error
  return {'ok': l:code == 0, 'code': l:code == 0 ? '' : 'exit', 'error': l:code == 0 ? '' : 'exit ' . l:code,
        \ 'exit_code': l:code, 'stdout': l:out}
endfunction
```

### `test/.themisrc`

```vim
scriptencoding utf-8
let s:root = fnamemodify(expand('<sfile>:p:h:h'), ':p')
execute 'set runtimepath^=' . fnameescape(s:root)
set nomore
set encoding=utf-8

" Pred() が真になるまで 10 ms ずつ待つ。:sleep 中に job コールバックが走る（両エディタ）
function! ${PluginName}TestWaitUntil(Pred, timeout_ms) abort
  let l:start = reltime()
  while !a:Pred() && reltimefloat(reltime(l:start)) * 1000 < a:timeout_ms
    sleep 10m
  endwhile
  return a:Pred()
endfunction

" 隔離された環境: g: を全部消し、データファイルを一時ディレクトリへ、時刻を固定
function! ${PluginName}TestReset() abort
  for l:name in keys(g:)
    if l:name =~# '^${plugin_name}_'
      execute 'unlet! g:' . l:name
    endif
  endfor
  let l:dir = tempname()
  call mkdir(l:dir, 'p')
  let g:${plugin_name}_file = l:dir . '/data.json'
  let g:${plugin_name}_test = {'now': 1700000000}
  " テスト開始前から存在するタイマー（themis 等のもの）は止めない
  let s:timers_before = map(timer_info(), 'v:val.id')
  return l:dir
endfunction

" 後片付け: ファイル、バッファ、自分の augroup、テスト中に作ったタイマー、s: 状態
function! ${PluginName}TestCleanup(dir) abort
  call setfperm(a:dir, 'rwx------')
  call delete(a:dir, 'rf')
  for l:b in getbufinfo()
    if l:b.name =~# '^${plugin_name}://'
      execute 'bwipeout!' l:b.bufnr
    endif
  endfor
  " テスト中に作られたタイマーだけ止める（timer_stopall() は他のものも止める）。
  " テスト中に他のコードがタイマーを作らない（-u NONE、themis はタイマーを使わない）前提の簡易方式。
  " 厳密にやるなら compat 側で自作タイマーの ID を管理する
  for l:t in timer_info()
    if index(s:timers_before, l:t.id) < 0
      call timer_stop(l:t.id)
    endif
  endfor
  call ${plugin_name}#app#store#_reset()
endfunction
```

### テストファイルの形

```vim
scriptencoding utf-8
let s:suite = themis#suite('app_item')
let s:assert = themis#helper('assert')

function! s:suite.before_each() abort
  let s:dir = ${PluginName}TestReset()
  let g:${plugin_name}_test.cmd_runner = function('s:runner')
endfunction

function! s:suite.after_each() abort
  call ${PluginName}TestCleanup(s:dir)
endfunction

function! s:runner(argv, opts) abort
  return {'ok': v:true, 'code': '', 'error': '', 'exit_code': 0, 'stdout': ''}
endfunction

function! s:suite.add_then_find() abort
  let l:r = ${plugin_name}#app#item#add('https://a.example/', 'A', v:null, 1000)
  call s:assert.true(l:r.ok, string(l:r))
  call s:assert.equals(${plugin_name}#app#store#find(${plugin_name}#app#store#current(), l:r.id).display_name, 'A')
endfunction
```

* 注入点は `g:${plugin_name}_test` の Dictionary 1 つ（`cmd_runner`, `http_fetcher`, `now`, `prompt`）。グローバルの Funcref 変数は大文字始まりが必須（E704）なので Dictionary に入れる。各注入点は型を見てからだけ使う。本番の vimrc で設定されても動く設計なので、help の Testing hooks に「利用者は設定しない」と書き、Funcref を受ける注入点は `type() == v:t_func` を確認してから呼ぶ
* テストの分類：`domain_*`（unit。Vim の機能を使わない）/ `storage_*`（integration。一時ディレクトリに実際に書く）/ `app_*`（integration。store は実ファイル、deps は注入）/ `deps_*`・`compat_*`（contract と実ジョブ。実ジョブは Unix 前提）/ `ui_*`（integration。バッファを実際に開き、プロンプトは注入）/ `plugin_*`（コマンド引数）/ `e2e_*`（通し）
* `themis#helper('scope')` で `s:` 関数を直接呼べる。外部 UI を起動せずに sink をテストするときに使う
* `-e -s`（Ex モード + サイレント）では画面描画が無いためシンタックスが評価されない（`synID()` は 0）。ハイライトの検証は `matchaddpos` の id 数や `execute('syntax list')` のパターンで行う
* テストは順序を入れ替えても単独でも通ること。`before_each` で残存（自分のバッファ・タイマー・ジョブ・一時ディレクトリ）が無いことを assert すると、後片付けの漏れが次のテストの失敗として見える
* 失敗注入（`cmd_runner` / `http_fetcher` が失敗や遅延を返す）と、ジョブの完了・停止・上限超過の順序を入れ替えたテストを持つ
* `:helptags doc` をテストで実行し、重複タグ（E154）が無いこと

### doc/*${plugin_name}.txt*

必ず置く節：Introduction / Quick start / Commands / The buffer（keys, mappings, overridden keys）/ Options（各オプションに `*g:${plugin_name}_xxx*` タグと既定値）/ Functions / The data file（形式と場所）/ **Privacy**（何を・いつ・どこへ送るか、止め方）/ Highlights / **Limitations**（競合検出の分解能、Windows、戻らない外部コマンド、既知の罠）/ Testing hooks（`g:${plugin_name}_test` は内部インターフェース）。末尾は `vim:tw=78:ts=8:ft=help:norl:`。`doc/tags` はコミットしない。

```
==============================================================================
1. 概要                                                       *${plugin_name}-intro*

${plugin_name}
(何ができるかを 2〜3 行で)

要件:
- Vim: 9.0+（必要なパッチがあれば併記）
- Neovim: 0.9+
- 外部コマンド: curl（任意。無いとき省略される機能を書く）

==============================================================================
2. 機能                                                     *${plugin_name}-usage*

:${PluginCommand} [how]                                      *:${PluginCommand}*
        説明: (何をするコマンドか)

${plugin_name}#open([{how}])                             *${plugin_name}#open()*
        説明: 公開 API（autoload/${plugin_name}.vim に集約）

*g:${plugin_name}_open*
g:${plugin_name}_open (default: 'vsplit')
        'vsplit' / 'split' / 'edit' / 'tab'

------------------------------------------------------------------------------
vim:tw=78:ts=8:ft=help:norl:
```

### Makefile

* git clone した後に、バージョンを固定するために Makefile を更新すること（THEMIS_COMMIT, VIMLPARSER_COMMIT, VIMLINT_COMMIT。取得元・確認日・動作確認した Vim/Neovim の版を記録する）。固定後は `make pin` で既存の clone も同じ commit に揃える
* `-e -s` が無いと tty 無しの themis が固まる。`</dev/null` で CI と同じ stdin 条件にする
* `-c func_abort=1` は付けない（固定版の vimlint はこれで常に exit 1 になる。全関数に `abort` を付ける規約で代替）
* `EVL103`（未使用引数）は無効にする（job / timer / autocmd のコールバック引数はエディタが決める）
* 固定版の vimlint はラムダ（`{a, b -> …}`）で落ちる。名前付き関数と partial を使う
* lint の対象にするディレクトリは要調節（autoload plugin test）

```
.DEFAULT_GOAL := help
SHELL := /bin/bash

PWD := $(shell realpath $(dir $(lastword $(MAKEFILE_LIST))))
DEPS_DIR := $(PWD)/.deps

.PHONY: init deps pin tester linter lint test test-vim test-nvim help clean
.PHONY: lint-vint vint-int check-vint

THEMIS_URL    := https://github.com/thinca/vim-themis.git
THEMIS_DIR    := $(DEPS_DIR)/vim-themis
THEMIS_COMMIT :=

VIMLPARSER_URL    := https://github.com/ynkdir/vim-vimlparser.git
VIMLPARSER_DIR    := $(DEPS_DIR)/vim-vimlparser
VIMLPARSER_COMMIT :=

VIMLINT_URL    := https://github.com/syngan/vim-vimlint.git
VIMLINT_DIR    := $(DEPS_DIR)/vim-vimlint
VIMLINT_COMMIT :=

LINT_DIRS ?= autoload plugin test

define CLONE_AND_OPTIONAL_PIN
	@set -euo pipefail; \
	url="$(1)"; dir="$(2)"; commit="$(3)"; \
	mkdir -p "$(DEPS_DIR)"; \
	if [ ! -d "$$dir/.git" ]; then \
		git clone "$$url" "$$dir"; \
	fi; \
	if [[ "$$commit" =~ ^[0-9a-fA-F]{7,40}$$ ]]; then \
		git -C "$$dir" checkout -q --detach "$$commit"; \
		[[ "$$(git -C "$$dir" rev-parse HEAD)" == "$$commit"* ]] \
			|| { echo "error: $$dir HEAD does not match $$commit"; exit 1; }; \
	fi; \
	echo "OK: $$dir @ $$(git -C "$$dir" rev-parse --short HEAD)"
endef

init: linter tester ## init

deps: tester linter ## clone deps (optionally pin)

pin: deps ## checkout pinned commits in existing clones (fails if a COMMIT is empty or malformed)
	@for c in "$(THEMIS_COMMIT)" "$(VIMLPARSER_COMMIT)" "$(VIMLINT_COMMIT)"; do \
		[[ "$$c" =~ ^[0-9a-fA-F]{7,40}$$ ]] \
			|| { echo "error: set THEMIS_COMMIT / VIMLPARSER_COMMIT / VIMLINT_COMMIT to commit hashes first"; exit 1; }; \
	done
	$(call CLONE_AND_OPTIONAL_PIN,$(THEMIS_URL),$(THEMIS_DIR),$(THEMIS_COMMIT))
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLPARSER_URL),$(VIMLPARSER_DIR),$(VIMLPARSER_COMMIT))
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLINT_URL),$(VIMLINT_DIR),$(VIMLINT_COMMIT))

tester: $(THEMIS_DIR) ## clone tester

linter: $(VIMLPARSER_DIR) $(VIMLINT_DIR) ## clone linter

$(THEMIS_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(THEMIS_URL),$(THEMIS_DIR),$(THEMIS_COMMIT))

$(VIMLPARSER_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLPARSER_URL),$(VIMLPARSER_DIR),$(VIMLPARSER_COMMIT))

$(VIMLINT_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLINT_URL),$(VIMLINT_DIR),$(VIMLINT_COMMIT))

test-vim: tester ## testing with vim-themis (Vim)
	THEMIS_VIM=vim THEMIS_ARGS='-Nu NONE -U NONE -i NONE -n -e -s' \
		$(THEMIS_DIR)/bin/themis --reporter spec -r test </dev/null

test-nvim: tester ## testing with vim-themis (Neovim)
	THEMIS_VIM=nvim THEMIS_ARGS='--headless -u NONE -i NONE -n' \
		$(THEMIS_DIR)/bin/themis --reporter spec -r test </dev/null

test: test-vim test-nvim ## testing with vim-themis (Vim and Neovim)

lint: linter ## linting with vimlint
	$(VIMLINT_DIR)/bin/vimlint.sh \
		-l $(VIMLINT_DIR) \
		-p $(VIMLPARSER_DIR) \
		-e EVL102.l:_=1 -e EVL103=1 \
		$(LINT_DIRS) 2>&1

lint-vint: check-vint ## linting with vint
	vint plugin autoload test

vint-int: ## install vint using pip
	pip install vint

check-vint: ## check vint command exists
	command -v vint >/dev/null 2>&1

clean: ## remove cloned deps
	rm -rf $(DEPS_DIR)

help: ### help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
		| sort \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
```

### README.md

* # ${PluginName}
* ## Installation（vim-plug とネイティブパッケージ。最低版と必要な外部コマンド）
* ## Features
  * ### Commands
  * ### Keys（専用バッファのキー。上書きするキーを明記）
  * ### Functions
  * ### Options（`| option | default | meaning |`。help と既定値を一致させる）
* ## Configuration
  * ### Key Mappings (Example)
* ## Privacy
* ## Limitations
* ## Development（`make deps && make test && make lint`）
* ## LICENSE

### LICENSE ファイルの用意

* 2026 は、現在の年
* @kis9a は、現在の GitHub のユーザー名に変更すること
* 他プラグインからコードを移植するときは、移植元のライセンスが要求する著作権表示とライセンス文を保つ（MIT なら表示の保持が条件）。移植元・関数名・コミットをファイル先頭のコメントに書き、別作者のコードなら `THIRD_PARTY_NOTICES` にも載せる

```
The MIT License (MIT)

Copyright (c) 2026 @kis9a

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### .gitignore に含めるもの

* gitignore.io 等で Vim を含める
* 追加で以下を入れる（生成物/依存）
  * `.deps/`
  * `tmp/`
  * `.mcp.json`
  * `CLAUDE.md`
  * `doc/tags`

---------

## 品質担保

### 目的

* Vim / Neovim 両方で同じテストを回し、互換性を継続的に保証する
* テストは ユーザー環境に依存しない（`-u NONE` / `--headless` などで固定）
* 利用者の実際の設定でも壊れないことを、pty で確認する

### 依存取得

```sh
make deps
```

### lint（静的チェック）

```sh
make lint
make lint-vint
```

### 自動テスト（Themis）

```sh
make test-vim
make test-nvim
make test        # 両方
```

CI では「宣言した最低版」と「最新安定版」の両方を Vim / Neovim それぞれで回す（`THEMIS_VIM` にバイナリを渡す）。Vim は `+job` `+channel` `+timers` などの機能ビルドも確認する。

テストの絞り込み（特定ファイルだけ）

```sh
./.deps/vim-themis/bin/themis --reporter spec -r test test/<file>.vim
```

### pty E2E

実際の端末でキーを打つ。macOS / BSD は

```sh
TERM=xterm-256color script -q /dev/null vim -Nu NONE -i NONE -n +'set rtp^=.' +'runtime plugin/${plugin_name}.vim'
```

Linux（util-linux）は `script -q -c 'vim -Nu NONE …' /dev/null`、または Python の `pty` で統一する。`-u NONE` と利用者の実 vimrc の両方で行い、`:messages` に `E\d+` が無いこと。プロンプトが開いている間に意図しないキー（撮影コマンドなど）を送らない。

### 手動テスト

Vim

```sh
vim -Nu NONE -U NONE -i NONE -n +'set rtp^=.' +'runtime plugin/${plugin_name}.vim'
```

Neovim

```sh
nvim -u NONE -i NONE -n +'set rtp^=.' +'runtime plugin/${plugin_name}.vim'
```

---------

## 互換性メモ

設計や実装で前提にしてよい事実。「根拠」が help タグのものは公式仕様、「実測」のものは Vim 9.2（patch 1-750）/ Neovim 0.12 / macOS で観測した挙動で、他の版や OS では確かめ直す。ここに無いものは help を読むか、`vim -es` / `nvim --headless` で試して確かめる。記録するときは「主張 / 対象（エディタ・版・OS）/ 根拠 / 検証状態 / 設計への影響」を揃える。

| 事実 | 影響 | 根拠 |
|---|---|---|
| `system(List)` / `systemlist(List)` は Vim 9.2.0250 から。それ以前は E730 | `has('nvim') \|\| has('patch-9.2.250')` で判定 | `:help E1575`、patch 9.2.0250、実測（E730） |
| `system()` の出力には stderr が混ざる（String / List とも） | stderr を分けるなら job | 実測（両エディタ） |
| Vim の `system(List)` は不存在コマンドで例外を出さず `v:shell_error` 127。Neovim は E475 を投げる | 先に `executable()` | 実測 |
| Vim の `system(List)` は stdout を握った孫プロセスの終了まで待つ | 起動だけしたいコマンドは job で | 実測（`sh -c 'sleep 3 &'` で 3 s） |
| Neovim の `json_encode` は非 UTF-8 文字列で E474、`json_decode` も拒否。Vim は不正バイトをそのまま書く | 保存前に UTF-8 検査。片方で書いたファイルを他方が読めなくなる | 実測（E474） |
| 正規表現・`trim()`・`substitute()`・`split('\zs')` は基底文字と結合文字をまとめて 1 文字として扱い、どれを結合文字とみなすか（タグ文字、ZWJ）が Vim と Neovim で違う | 名前の無害化はコードポイントの List 上で。両エディタで結果が一致しないと、片方で保存した名前をもう片方が拒否する | 実測 |
| 正規表現は不正な 1 バイト（0x82）を U+0082 として読む | 正規表現では不正 UTF-8 を検出できない。`str2list()` 往復で検査 | 実測 |
| Neovim の正規表現は文字集合の範囲で U+10000 以上を扱えない（`[\U000e0000-…]` が効かない） | タグ文字などは List 上で範囲比較 | 実測 |
| over-long UTF-8 やサロゲートを Vim は 1 文字として受け付け、Neovim の `json_decode` は拒否する | 保存前に拒否 | 実測 |
| `%delete _` + `setline` は fold 状態を消す。既存行への `setline` 上書きは残す | 再描画は上書き | 実測 |
| `nomodifiable` のバッファに `setline()` すると E21 | 描画中だけ `modifiable` | `:help 'modifiable'` |
| job / timer のコールバックは `input()` 中にも走る | 再描画を遅延判定 | 実測 |
| コールバック内では `state()` が常に `c` を含む。themis 下では `oxS` | `state()` を遅延条件にしない | 実測（`:help state()`） |
| `writefile(…, 'b')` は末尾改行を付けない | `'b'` を付けない | `:help writefile()` |
| `input()` の `<Esc>` は空文字、例外は CTRL-C のみ。Neovim は `cancelreturn` 可 | 「空 = 取消」の仕様か `cancelreturn` | 実測、`:help input()` |
| 利用者の `cnoremap` は `input()` にも効く | CTRL-C 以外の取消経路を残す | `:help input()`（コマンドラインとして扱われる） |
| tty 無しの `input()`：Vim `-es` はスクリプト打ち切り、Neovim headless は無期限ブロック。`confirm()` は既定値を返す | `has_tty()` ガード | 実測、`:help confirm()` |
| 端末から起動した `nvim --headless` でも `has('ttyin')` は 1 | Neovim は `nvim_list_uis()` で判定 | 実測 |
| `inputlist()` / `getchar()` は tty 無しでブロック | 同上 | 実測 |
| グローバル Funcref 変数は大文字始まり必須（E704） | 注入点は Dictionary | `:help E704` |
| `nvim_notify()` は Neovim 0.11 以降非推奨 | `echohl` + `echomsg` | Neovim `deprecated.txt` |
| Neovim `jobstart()` は不存在コマンドで E475 を表示して -1 | 先に `executable()` | 実測、`:help jobstart()` |
| Vim `job_status()` は Unix では不存在コマンドでも `'dead'`（exit 122）、`'fail'` にならない | `fail` 分岐は Windows 用 | `:help job_status()` |
| Vim の `exit_cb` は最後の `out_cb` より先に来得る。`close_cb` が出力の終わり | 両方待つ | `:help job-exit_cb`、`:help close_cb`、実測（300 KB） |
| Neovim `stdout_buffered` は EOF まで `on_stdout` を呼ばない | 出力上限と両立しない | `:help channel-buffered` |
| Neovim の `on_stdout` は main loop が詰まると 1 回で何 MiB も渡す | `join` 前に切る、`chanclose` | 実測（1 回で 16 MB） |
| `mapcheck(key)` は接頭辞の一致（`qr` があると `q`）でも非空 | 同じ lhs は `maparg()` で | `:help mapcheck()`、実測 |
| バッファローカル `<nowait>` マップはグローバルの接頭辞マップに勝つ | 既定キーは `<nowait>` | `:help :map-<nowait>` |
| `setlocal` のウィンドウローカルオプションは現在窓のみ。同じバッファの `:split` は `BufWinEnter` を起こさない | 全窓 `win_execute` + `WinEnter <buffer>` | `:help local-options`、実測 |
| `BufWinLeave` は同じバッファが他の窓に表示されていると発火しない。`matchaddpos` のマッチは窓に残る | `BufLeave` + `timer_start(0)` で窓を見直して掃除 | 実測 |
| `getwinvar()` / `winnr()` は現在のタブの窓しか見えない | 全タブは `win_findbuf()` と `win_execute()` | `:help getwinvar()` |
| `matchaddpos()` / `clearmatches()` はウィンドウ単位 | id を `w:` に保持 | `:help matchaddpos()` |
| `glob()` はパス中の `[` `*` をパターンとして解釈 | 掃除は `readdir()` + 完全一致 | `:help glob()` |
| `mkdir(…, 'p')` の失敗は例外（実測 E739）、`writefile` は -1 を返す経路と例外（実測 E482）の両方がある | `try` で包み、戻り値も見る | 実測、`:help writefile()` |
| 既存ファイルへの `writefile()` は権限を保つ | 空ファイル → `setfperm` → 内容の順で書く | 実測（両エディタ） |
| `writefile(…, 's')` は書いた後に `fsync()` を呼ぶ。既定は `'fsync'` オプションに従う | 永続性を上げたい保存に | `:help writefile()` |
| `bufload()` は未ロードのバッファで `BufReadCmd` / `BufEnter` / `BufWinEnter` を起こし得る。`setbufvar` のオプションは未ロードでも効く | オプションはロード前に付ける | 実測（両エディタ）、`:help bufload()` |
| Neovim で `chanclose(job, 'stdout')` した後は `on_stdout` の EOF（`['']`）は来ない | 自分で閉じたら `on_exit` だけで完了 | 実測 |
| `setfperm()` は非 0 が成功、0 が失敗。`mkdir()` は 1 が成功、既にあれば例外（E739） | 戻り値の向きを間違えない。`mkdir` の原子性はロックに使える | `:help setfperm()`、`:help mkdir()`、実測 |
| `getfsize()` は見つからないと -1、Number に収まらないと -2、ディレクトリは 0 | 負値も拒否 | `:help getfsize()` |
| Neovim の `on_exit` は stdout / stderr の EOF（`['']`）の後に来る | help は順序を約束しないので両方待つ | 実測、`:help channel-callback` |
| `rename()` は同一ファイルシステム内で原子的、既存を上書き | tmp を同じディレクトリに | `:help rename()`（永続性・Windows は保証外） |
| Dictionary / List 内では `1 == 1.0` が偽 | Float を正規化 | 実測 |
| `printf('%.4f')` は `0.00004` を `0`、`1e300` を 300 桁に。Vim の `printf('%g')` は末尾の 0 を消さない | Float の表現は自前で整える | 実測 |
| `:sleep` 中に job コールバックが処理される（両エディタ） | テストの待ちに使える | `:help :sleep`、実測 |
| `acwrite` + `BufWriteCmd <buffer>` は `:w` `:wq` `:x` `:update` で発火（未変更の `:x` `:update` は通らない）。`modified` を落とさないと `:wq` は閉じない | 編集モードの `:w` | `:help BufWriteCmd`、実測 |
| `BufWriteCmd` の中ではバッファ内容を変えない（help の要求）。`modified` を落とすのはハンドラの責務 | 成功後の再描画は `timer_start(0)` で後に | `:help BufWriteCmd` |
| `:e!` は acwrite でも `BufReadCmd` を発火し、`b:` 変数は残る | 破棄の実装 | 実測 |
| `setlocal undolevels<` で実効値がグローバルに戻る（両エディタ） | 編集モードで undo を戻す | `:help :setlocal`、実測 |
| `setline()` は同じ文字列でも `modified` を立てる | 描画後に `setlocal nomodified` | 実測 |
| `matchadd('Conceal', pat, …, {'conceal': ''})` + `conceallevel=2` で完全に隠れる | `syntax` 無しで conceal | `:help matchadd()`（`conceal` 引数）、実測 |
| `fzf#run` を `autoload/fzf.vim` 以外で定義すると E746 | テストのスタブは一時ディレクトリに `autoload/fzf.vim` を置いて rtp に足す | `:help E746` |
| themis は `-e -s` でシンタックスを評価しない（`synID()` が 0） | ハイライトのテスト方法 | 実測 |
| vim-plug の `on:` 遅延ロード中はそのプラグインの autoload 関数が E117 | 他プラグインを実行時に呼ばない。利用者のマッピングはコマンドへ | `plug.vim` の `s:reorg_rtp()` を読んで確認 |
| 自作の pty ハーネスが端末の問い合わせ（DSR / OSC）に応答しないと Neovim が `E1568` を出す | プラグインの不具合ではない。`:messages` の判定から除外 | 実測 |
| `vim -es` の `-c` は `-bar` 無しコマンドの後ろの `\|` を引数として渡す | スクリプトは `-S` で | `:help -c`、実測 |
| `expand(path)` は `%` `#` も展開する | パス解決は `fnamemodify(substitute(path, '^\~', expand('~'), ''), ':p')` | `:help expand()` |
| curl の `--max-filesize` は 8.4.0 より前では事前にサイズが分からない転送に効かない | job 側のバイト上限を必須に | curl の man |
