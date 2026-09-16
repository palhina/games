# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

mopsworks.jp 向けの小さなブラウザゲーム集。ビルド手順・パッケージマネージャ・依存関係・テストは一切なく、各ゲームは自己完結した1枚の `index.html`(HTML + CSS + JS をすべてインライン)と、その隣の `img/` フォルダだけで構成されている。このディレクトリ自体が git リポジトリで、`github.com/palhina/games`(private / 既定ブランチ `main`)に push している。公開サイトは兄弟ディレクトリの `../HP` リポジトリ(GitHub Pages / `mopsworks.jp`)にあり、ゲーム内の絶対 URL(`og:image` → `https://mopsworks.jp/games/nigeru/cup/img/og.png`)から、このツリーがそのまま `/games/` 配下で配信されることが分かる。それ以外の参照はすべて相対パス。**この方針を崩さないこと。**

```
README.md           リポジトリの説明
index.html          ゲーム一覧(ゲームを追加したらここにリンクを足す)
nigeru/index.html   cup/ への meta-refresh リダイレクト(+ `file://` 用のフォールバック)
nigeru/cup/         コップ、逃げる。(逃げる。シリーズ 01)— 現時点で唯一のゲーム
```

## 動かし方

```bash
python3 -m http.server 8000     # このディレクトリで実行し、http://localhost:8000/nigeru/cup/ を開く
```

`file://` ではなく HTTP で配信すること — `img/nya.mp3` は `fetch()` + `decodeAudioData` で読み込んでおり、`file://` では失敗する。

さらに `file://` ではディレクトリ URL(`cup/`)が index.html ではなくファイル一覧になる。そのため `index.html` と `nigeru/index.html` は、`location.protocol === 'file:'` のときだけリンク先に `index.html` を補う1行を持っている。公開サイト(HTTP)では従来どおり `cup/` に飛ぶので、`mopsworks.jp/games/nigeru/cup/` という正規 URL は維持される。**画像が出ない・一覧が出ると言われたら、まず `file://` で開いていないかを疑う。**

- `?t=55` で 55 秒地点から開始できる(`START_T`)。終盤の難易度、45 秒以降の死亡セリフ、60 秒クリアを確認する現実的な唯一の手段。
- ベストタイムは `localStorage['nekocup_best']`(`STAGE.storageKey`)。初回起動状態を試すときは消す。
- 音声はユーザー操作が必要。最初の pointerdown /「はじめる」クリックで `audioInit()` が走る。

## ゲームの構造(`nigeru/cup/index.html`)

**`STAGE` がコンテンツをすべて持ち、その下のエンジンはステージ非依存。** clearTime、難易度テーブル、攻撃のタイミング、猫のセリフ、称号、机の小物、ストレージキーはすべてこの1つのオブジェクトにある。シリーズ2作目は「同じエンジン + 別の `STAGE`」を新しいフォルダに置く想定。挙動を変えるときは、値を直書きせず `STAGE` にフィールドを足す方を優先する。

**攻撃はステップ配列。** 各ビルダー(`attackA`〜`attackD`)は `[{ d: 秒数, run(p, ctx) }, ...]` を返す。`p` はそのステップ内を 0→1 で進む進捗、`ctx` はその攻撃1回分の共有状態(狙いを定めた x など)。`stepAttack()` が毎フレーム進める。攻撃を追加する手順は、ビルダーを書く → `ATTACKS` に登録 → 難易度テーブルのどれかの `types` にキーを追加、の3つ。`pickType()` は、新しく解禁された型を必ず1回目に出し、以降は新しい順に 1.6 / 1.2 / 1.0 の重みを付け、同じ型の3連続を禁止する。

**座標はすべて `layout()` から導出される。** `W` / `H` は `#game` ボックス(max-width 440px・縦持ち・`overflow:hidden`)のサイズ。机の奥端は H の 44%、コップの底は 73%、`cupLine` は肉球の中心が来る高さ、肩 `SH.L/R` は机の縁のすぐ下。`layout()` はリサイズ時に再実行されるので、新しく配置する要素も固定 px ではなく W/H の比率として `layout()` の中に書く。

**腕**は、肩を transform-origin に固定した棒を回転させたもの。`armTo(side, tx, ty, depth)` が `(tx, ty)` に向けた角度と `depth × 距離` の高さを設定し、`depth = 1` のとき肉球の先端がちょうど目標に届く。

**当たり判定は横方向1次元のみ** — 着地する肉球には `hitAt(tx)`、攻撃 C の薙ぎ払いには `hitSweep(x0, x1)` を使い、どちらも `HIT_DIST = PAW_W/2 + CUP_W/2 - HIT_MARGIN` と比較する。難易度は絵のサイズをいじるのではなく、`HIT_MARGIN` と予告 / 突きの時間で調整する。

**猫は PNG + SVG オーバーレイ。** `face_closed.png` / `face_open.png` を `.nya` クラスで切り替え、瞳孔と寝顔だけをインライン SVG で上に重ねている。瞳孔の座標(`EYE.l`、`EYE.r`、`STAGE.pupil.*`)は画面 px ではなく **SVG の 860×565 画像座標**。そして「予告の見せ方」がこのゲームの本体:待機中は瞳孔がコップを追い、狙っている間は細くなり、突きの `nyaLead` 秒前に丸く見開いて「にゃっ」と鳴く。予告音は `sfx()` で意図的に鳴らしていない — 目だけが唯一の合図。

**効果音**は WebAudio による合成(`tone` / `noise`)。鳴き声だけは `img/nya.mp3` のデコード結果を優先し、読み込めなければ合成音にフォールバックする。

**入力**は 1.15 倍の係数を掛けたポインタードラッグと、16ms の `setInterval` でポーリングする矢印キー。

## 決めごと

コード中のコメントとプレイヤーに見えるテキストはすべて日本語。猫のセリフは意図的に関西弁で淡々とした調子にしている。シリーズの新作も1ファイル完結・依存関係なしを維持すること。
