# 266-nekotsumi「ねこつみ」仕様書

最終更新: 2026-09-27

## 概要
屋上のダンボール箱にねこを積み上げる1タップ積みゲーム。重心が箱からはみ出すと崩落。
ときどき混ざる「いぬ」を積むと全員が驚いて総崩れになるので、よけて落とすのが正解。

- エントリ: `index.html`（`nekotsumi.html` へリダイレクト。旧URL維持のため index は消さない）
- 実体: `nekotsumi.html` 単一ファイル（外部JSなし・フォントのみGoogle Fonts）
- 論理解像度: 400×可変。canvasは `#wrap`(max-width 560px) に全面配置

## 操作／設定UI（コントロールパネル）

| 項目 | 仕様 |
|---|---|
| 入力方式 | Pointer Events 一本化。canvasタップ＝落とす／キーボードは Space・Enter |
| 仮想パッド | なし（1タップゲームのため画面全体が入力面） |
| ボタン | `#start`「つむ」／`#retry`「もう一度」／`#mute`「音：オン/オフ」。いずれも `tapBtn()` 経由で pointerdown + 400ms ロック + `navigator.vibrate(14)` |
| ポーズ | 明示ポーズなし。`visibilitychange` で BGM のみ自動停止・復帰 |
| ミュート | 右下固定 `#mute`（64×48px以上・safe-area下余白込み）。BGMと効果音の両方を落とす |
| localStorage | `tg.266.mute`（ミュート状態）／`nekotsumi_best`・`nekotsumi_top`（既存キー。セーブ互換のため改名しない） |

## 音

| 種類 | 実装 | 素材 |
|---|---|---|
| プレイ中BGM | HTMLAudio ループ・0.34まで50msごとにフェードイン | `Kina_Takes_the_Lead.m4a` |
| ゲームオーバーBGM | 同上。プレイ中BGMは停止して切替 | `Queen_of_the_Living_Room.m4a` |
| 効果音 | Web Audio 合成（鳴き声・ぴったり音・吠え・落下音） | なし |

- 両BGMとも `preload="none"`。初回再生はユーザー操作（「つむ」タップ）起点なので iOS の自動再生制限に抵触しない
- 素材は 192kbps MP3 を HE-AAC 64kbps m4a へ変換（8.66MB → 2.87MB）

## iOS対応

- `touch-action`: body = `manipulation` / canvas = `none` の2段構え
- ダブルタップ防止: 350ms以内の連続 `touchend` を `preventDefault` ／ `dblclick` ・ `gesturestart/change/end` ・ `contextmenu` を無効化
- `user-scalable=no, viewport-fit=cover` ／ safe-area は `:root` と `.overlay` `#mute` で確保
- DPRは最大2でクランプ

## ゲームルール

| 要素 | 値 |
|---|---|
| ねこ4種 | こねこ 62×44 / ねこ 88×56 / でぶねこ 118×66 / ながねこ 142×48 |
| いぬ | 50×58（丸胴）。stack 2体以上で15%の確率で出現 |
| 箱 | x=110〜290、上端 y=70 |
| 移動速度 | 80 + 積み数×7（上限280） |
| 崩落判定 | 各段から上の重心が、その下の段の接地幅を外れたら崩落 |
| いぬ | 重なって積むと全員驚いて総崩れ／重なり0でよけると「セーフ！」 |
| ぴったり | 直下との中心差4px未満でコンボ加算 |
