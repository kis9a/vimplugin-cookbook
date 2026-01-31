# vimplugin-cookbook

## 概要

このドキュメントは、Vim / Neovim 両対応プラグインを「長く壊れにくく」開発・保守するための共通ガイドです。
設計（アーキテクチャ）、実装手順、互換性ポリシー、性能/安全性のチェック観点を整理し、実装のブレ・互換性事故・起動/実行の性能劣化を防ぐことを目的とします。

## 前提

Vim / Neovim 両対応プラグインとして実装する

### ルール

* 基本は レガシー Vimscript（Vim9script 依存は禁止）
* バージョン判定より 機能検出：`has`, `exists`, `executable()` を使用
* Vim/Neovim 固有 API（job/channel/UI 等）は compat 層に隔離し、呼び出し側は共通 API のみ使用
* **外部依存（外部コマンド / 他プラグイン / 外部ファイル・環境）** は deps（adapter）層に隔離し、呼び出し側は deps の共通 API のみ使用
* 機能が無い環境向けに必ず フォールバック（例：非同期→同期 `system`）を用意
* 構成は標準 runtime（`plugin/`, `autoload/`, `doc/`）。名前空間で汚染を避ける
* `plugin/` から重い処理を呼ばない（autoload 遅延ロードを徹底：コマンド/augroup定義のみ）
* スクリプトに非ASCII（日本語メッセージ等）を含めるなら `scriptencoding utf-8` を明示（文字化け事故防止）

### 参照許可

* Vim/Neovim の公式 help を根拠にしてよい（context7 参照も可能）
* context7 / serena mcp の利用を許可

### 出力

* 実装（差分が分かる形）
* 互換性メモ（分岐点と理由、フォールバック）
* Vim/Neovim それぞれの簡単なテスト手順

### レガシー Vimscript を使う理由

* 両対応の「共通土台」として最も壊れにくい
* Vim/Neovim の差分が出る箇所（job/channel/UI 等）は **薄い compat 層**で吸収できる
* Vim9script は Neovim 非対応の前提があるため、両対応を最優先する限り避けるのが安全

---------

## 確認観点（レビュー/自己点検チェックリスト）

### パフォーマンス劣化懸念の払拭

* [ ] 起動時に重い処理をしていない（`plugin/` は軽い）
* [ ] FileType などで対象を絞れている（全バッファ常時は避ける）
* [ ] 連続イベント（CursorMoved 等）を多用していない
* [ ] キャッシュ戦略（`s:`/`b:`）がある or 不要と説明できる
* [ ] 非同期が使えない場合の同期フォールバックが安全に動く

### セキュリティチェック

* [ ] `system()` / `:execute` に **ユーザー入力を直結しない**
* [ ] 外部コマンド引数は `shellescape()` などで保護
* [ ] 外部コマンドは **deps 層に隔離**し、許可するコマンド/サブコマンドが説明できる（無制限実行を避ける）
* [ ] 可能なら「引数リスト形式」や固定テンプレで組み立て、文字列連結を最小化する
* [ ] 任意文字列 `eval()`、無制限 `execute()` を避ける（使うなら制約明記）
* [ ] 勝手にネットワーク通信しない（するなら opt-in）

### 権限/影響範囲が強すぎないか

* [ ] `:command` はプレフィックス付き（衝突しない）
* [ ] `autocmd` は group を作り、解除/更新しやすい
* [ ] グローバル変数汚染をしていない（`g:${plugin}_...` のみ）

### 公開関数の線引き（API設計）

* [ ] 公開 API は `autoload/${plugin_name}.vim` に集約
* [ ] 内部関数は `s:` or `autoload/${plugin_name}/internal.vim` 等に隔離
* [ ] README と help の「Functions」と実体が一致
* [ ] 外部依存の窓口は `autoload/${plugin_name}/deps/*` に集約され、core/command は deps API のみ呼ぶ

---------

## プラグインのアーキテクチャ

### 全体方針

* **入口（plugin/）**：コマンド、autocmd、設定読み込み、ロードガード
* **コア（autoload/）**：実処理、公開 API
* **compat 層（autoload/.../compat.vim）**：Vim/Neovim 差分吸収
* **deps（adapter）層（autoload/.../deps/）**：外部依存（外部コマンド / 他プラグイン / 外部資源）吸収
* **内部（autoload/.../internal/）**：非公開の実装詳細
* **doc/**：`:help` 用ドキュメント
* **test/**：Themis テスト

---------

### 推奨レイヤ構造

```
plugin/${plugin_name}.vim
  └─ コマンド定義 / autocmd 定義 / g:loaded ガード

autoload/${plugin_name}.vim
  └─ 公開 API（${plugin_name}#...）

autoload/${plugin_name}/command.vim
  └─ コマンド実装（引数解析/入力検証 → core 呼び出し）

autoload/${plugin_name}/core.vim
  └─ 純粋ロジック（可能なら副作用を減らす）

autoload/${plugin_name}/compat.vim
  └─ job/channel/UI/notify/path 等の差分吸収

autoload/${plugin_name}/deps/
  ├─ command.vim
  │   └─ 外部コマンド実行（同期/非同期・検出・安全な引数組み立て・スタブ注入点）
  └─ plugin.vim
      └─ 他プラグイン呼び出し（存在確認・呼び出し規約・失敗時の戻り値統一）

autoload/${plugin_name}/internal/*.vim
  └─ 非公開の補助
```

#### compat 層の責務

* job/channel（あれば非同期、なければ同期）
* 通知（`:echoerr` / `nvim_notify` 等）
* UI（popup/float 等、任意機能）
* パス・一時ファイル・OS差（必要なら）

#### deps（adapter）層の責務

* **外部コマンド**：`executable()` で可用性判定、引数の安全な構築、タイムアウト/失敗時の扱い統一
* **他プラグイン**：`exists('*foo#bar')` / `exists(':FooCommand')` などで存在判定し、呼び出しの成否を共通の戻り値に寄せる
* **外部資源**（任意）：ファイル/環境変数/ネットワーク（opt-in 前提）など「プラグイン外」を触る処理の境界
* **テスト容易性**：deps 層に **スタブ注入点**（runner 差し替え等）を用意し、Themis では外部依存なしで検証できるようにする

> 目安：compat は「エディタ差分」、deps は「プラグイン外の世界」。core は deps/compat を直接触らない。

---------

## 実装手順

0. 雛形を作る

* LICENSE / README.md / Makefile / `.gitignore` / `.mcp.json`
* `plugin/`, `autoload/`, `doc/`, `test/`

1. 要件入力（ユーザー）
   目的 / スコープ（やること・やらないこと）/ 期待UX をプロンプトで渡す。

2. 要件理解・質問（orchestrator）
   不明点だけ質問して前提を確定する（互換性・UI・安全確認・依存コマンド等）。

3. 実装計画を作成（orchestrator）
   `./tmp/plan-<plugin>.md` を作る（以下を含める）

   * 提供する Commands / Functions / Options
   * ファイル構成（plugin/autoload/compat/doc/test）
   * Vim/Neovim 差分点とフォールバック方針
   * テスト方針（どこをThemisで検証するか）

4. 計画承認（ユーザー）
   承認後にのみ実装へ進む。

5. 実装（Task エージェント）
   コード変更は必ず Task に委任（必要なら並列可、同一ファイル同時編集は避ける）。

6. テスト（Task エージェント）
   テストを追加し、Vim/Neovim 両方で成功させる（`make lint` / `make test` など）。

7. ドキュメント更新（Task エージェント）
   README / `doc/<plugin>.txt` を実装と一致させる。

8. 提出物（Task → ユーザー）

   * 差分（変更点が追える形）
   * 互換性メモ（分岐点・フォールバック）
   * Vim/Neovim の簡単なテスト手順
   * チェックリスト（性能/安全性）の確認結果

---------

## 修正手順

1. 修正要件入力（ユーザー）：期待動作 / 実際 / 再現手順（可能なら）
2. 理解・質問（orchestrator）：不明点を潰す
3. 修正計画作成（orchestrator）：`./tmp/fix-plan-...md`（原因仮説・影響範囲・追加する回帰テスト）
4. 計画承認（ユーザー）
5. 修正実装（Task）：先に回帰テスト追加 → 修正 → lint/test 成功
6. ドキュメント更新（Task）
7. 確認観点チェック（Task）：性能/安全性/互換性のチェックリストを確認して提出

---------

## 実装ファイルについて

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
    command.vim
    core.vim
    compat.vim
    deps/
      command.vim
      plugin.vim
    internal/
      *.vim
test/
  ${test_aspect}.vim
```

### `plugin/${plugin_name}.vim` の基本形（入口を軽く）

```vim
if exists('g:loaded_${plugin_name}')
  finish
endif
let g:loaded_${plugin_name} = 1

" cpoptions guard（お作法）
let s:save_cpo = &cpoptions
set cpoptions&vim

" default options（必要なら）
if !exists('g:${plugin_name}_enabled')
  let g:${plugin_name}_enabled = 1
endif

command! -nargs=* ${PluginCommand} call ${plugin_name}#command#run(<q-args>)

let &cpoptions = s:save_cpo
unlet s:save_cpo
```

### `autoload/${plugin_name}.vim`（公開 API を集約）

> **`cpoptions` ガード**: autoload ファイルでも `<SID>` マッピングや `line-continuation`（`\` 行継続）を使う場合は `plugin/` と同様に `let s:save_cpo = &cpoptions | set cpoptions&vim` … `let &cpoptions = s:save_cpo` で囲むこと。

```vim
function! ${plugin_name}#run(input) abort
  return ${plugin_name}#core#run(a:input)
endfunction
```

### `autoload/${plugin_name}/compat.vim`（差分吸収の例）

```vim
function! ${plugin_name}#compat#has_nvim() abort
  return has('nvim')
endfunction

function! ${plugin_name}#compat#notify(msg, level) abort
  " level: 'info' | 'warn' | 'error'
  if exists('*nvim_notify')
    " Neovim（任意）
    call nvim_notify(a:msg, a:level ==# 'error' ? 4 : a:level ==# 'warn' ? 3 : 2, {})
  else
    if a:level ==# 'error'
      echohl ErrorMsg | echom a:msg | echohl None
    else
      echom a:msg
    endif
  endif
endfunction
```

### `autoload/${plugin_name}/deps/command.vim`（外部コマンド吸収の例）

```vim
" 戻り値は dict に統一（ok / code / stdout / stderr）
function! ${plugin_name}#deps#cmd#run(argv, opts) abort
  " argv: ['git','status',...] の想定（文字列連結しない）
  " opts: {'input': '', 'timeout_ms': 0} など（必要に応じて）
  if type(a:argv) != type([]) || empty(a:argv)
    return {'ok': v:false, 'code': -1, 'stdout': [], 'stderr': ['invalid argv']}
  endif
  if !executable(a:argv[0])
    return {'ok': v:false, 'code': -1, 'stdout': [], 'stderr': ['not executable: ' . a:argv[0]]}
  endif

  " テスト用スタブ注入点（Themis で外部依存なしにする）
  if exists('g:${plugin_name}_cmd_runner') && type(g:${plugin_name}_cmd_runner) == type(function('tr'))
    return call(g:${plugin_name}_cmd_runner, [a:argv, a:opts])
  endif

  let l:input = get(a:opts, 'input', '')
  let l:out = systemlist(a:argv, l:input)
  let l:code = v:shell_error
  return {'ok': l:code == 0, 'code': l:code, 'stdout': l:out, 'stderr': []}
endfunction
```

### `autoload/${plugin_name}/deps/plugin.vim`（他プラグイン吸収の例）

```vim
function! ${plugin_name}#deps#plugin#call(FuncName, args, opts) abort
  " FuncName: 'other#plugin#do'
  " args: [..]
  if type(a:FuncName) != type('') || empty(a:FuncName)
    return {'ok': v:false, 'value': v:null, 'error': 'invalid FuncName'}
  endif
  if !exists('*' . a:FuncName)
    return {'ok': v:false, 'value': v:null, 'error': 'missing dependency: ' . a:FuncName}
  endif
  try
    let l:v = call(a:FuncName, a:args)
    return {'ok': v:true, 'value': l:v, 'error': ''}
  catch /.*/
    return {'ok': v:false, 'value': v:null, 'error': v:exception}
  endtry
endfunction
```

### doc/*${plugin_name}.txt*

```
==============================================================================
1. 概要                                                       *${plugin_name}-intro*

${plugin_name}
(何ができるかを 2〜3 行で)

要件:
- Vim: 8.0+ (例)
- Neovim: 0.5+ (例)

==============================================================================

2. 機能                                                     *${plugin_name}-usage*

:${PluginCommand} [args]                                     *:${PluginCommand}*
        説明: (何をするコマンドか)
        例:   :${PluginCommand} foo

${plugin_name}#run({input})                              *${plugin_name}#run()*
        説明: 公開 API（autoload/${plugin_name}.vim に集約）
        例:   :echo ${plugin_name}#run('hello')

*g:${plugin_name}_enabled*
g:${plugin_name}_enabled (default: 1)
        1 のとき有効。0 で無効化。

------------------------------------------------------------------------------
vim:tw=78:ts=8:ft=help:norl:
```

### Makefile

* git clone した後に、バージョンを固定するために Makefile を更新すること (THEMIS_COMMIT, VIMLPARSER_COMMIT, VIMLINT_COMMIT)
* lint の対象にするディレクトリは要調節 (autoload plugin test)

```
.DEFAULT_GOAL := help
SHELL := /bin/bash

PWD := $(shell realpath $(dir $(lastword $(MAKEFILE_LIST))))
DEPS_DIR := $(PWD)/.deps

.PHONY: init deps tester linter lint test help clean
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
	fi; \
	echo "OK: $$dir @ $$(git -C "$$dir" rev-parse --short HEAD)"
endef

init: linter tester ## init

deps: tester linter ## clone deps (optionally pin)

tester: $(THEMIS_DIR) ## clone tester

linter: $(VIMLPARSER_DIR) $(VIMLINT_DIR) ## clone linter

$(THEMIS_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(THEMIS_URL),$(THEMIS_DIR),$(THEMIS_COMMIT))

$(VIMLPARSER_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLPARSER_URL),$(VIMLPARSER_DIR),$(VIMLPARSER_COMMIT))

$(VIMLINT_DIR):
	$(call CLONE_AND_OPTIONAL_PIN,$(VIMLINT_URL),$(VIMLINT_DIR),$(VIMLINT_COMMIT))

test: tester ## testing with vim-themis
	$(THEMIS_DIR)/bin/themis --reporter spec -r test

lint: linter ## linting with vimlint
	$(VIMLINT_DIR)/bin/vimlint.sh \
		-l $(VIMLINT_DIR) \
		-p $(VIMLPARSER_DIR) \
		-e EVL102.l:_=1 -c func_abort=1 \
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

* # ${Plugin-name}
* ## Installation
* ## Features
  * ### Commands
  * ### Functions
  * ### Options
* ## Configuration
  * ### Key Mappings (Example)
* ## LICENSE

### LICENSE ファイルの用意

* 2026 は、現在の年
* @kis9a は、現在の GitHub のユーザー名に変更すること

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
  * `vim-vimlint/`
  * `vim-vimlparser/`
  * `vim-themis/`
  * `tmp/`
  * `.deps/`
  * `.mcp.json`
  * `CLAUDE.md`
  * `docs/`（必要に応じて）

$ gitignore vim コマンドを使用可能

```
#!/bin/sh

gitignore() {
  api="gitignore.io/api/"
  case "$#" in
    0)
      curl -sL "$api/$(curl -sL "$api"/list | tr "," "\n" | fzf)" >>.gitignore
      ;;
    1)
      curl -sL "$api/$1" >>.gitignore
      ;;
    *) ;;
  esac
}

gitignore "$@"
```

## 品質担保

### 目的

* Vim / Neovim 両方で同じテストを回し、互換性を継続的に保証する
* テストは ユーザー環境に依存しない（`-u NONE` / `--headless` などで固定）

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

Vim

```sh
THEMIS_VIM=vim \
THEMIS_ARGS='-Nu NONE -U NONE -i NONE -n' \
make test
```

Neovim

```sh
THEMIS_VIM=nvim \
THEMIS_ARGS='--headless -u NONE -i NONE -n' \
make test
```

### テストの絞り込み（特定ファイルだけ）

```sh
./.deps/vim-themis/bin/themis --reporter spec -r test test/<file>.vim
```

### 手動テスト

Vim

```sh
vim -Nu NONE -U NONE -i NONE -n +'set rtp^=.' +'runtime plugin/<plugin>.vim'
```

Neovim

```sh
nvim -u NONE -i NONE -n +'set rtp^=.' +'runtime plugin/<plugin>.vim'
```
