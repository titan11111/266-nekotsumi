# 266-nekotsumi 学び

## 2026-09-27 いぬの縮小・丸胴化 ／ BGM実装 ／ ダブルタップ防止 ／ ハーネス通過

### やったこと

1. **いぬの横幅を100→50（50%）にし、胴を丸くした**
   - 角丸半径を `Math.min(h*.45,w*.38)` → `Math.min(w,h)/2` に上げて円に寄せた
2. **BGMを実装**（`Kina_Takes_the_Lead.m4a` = プレイ中 ／ `Queen_of_the_Living_Room.m4a` = ゲームオーバー）
3. **ダブルタップ防止**（touch-action 2段構え＋350ms連続タップ抑止＋gesture系無効化）
4. **ハーネス 8/8 PASS**

### 学び1: 「幅を半分にする」は幅の数値だけでは終わらない

`w:100→50` にしただけでは、**h基準で描いていた部位が胴からはみ出して、丸いどころか崩れた輪郭**になった。
実際にはみ出したのは3か所。すべて「h基準の固定値」だったため、wを半分にしても縮まなかった。

| 部位 | 元 | はみ出し量 | 直し方 |
|---|---|---|---|
| 垂れ耳の付け根 | `sg*h*.5` = ±29 | 胴半幅25を4px超過 | `Math.min(h*.5, w*.34)` |
| 耳の大きさ | `h*.14 × h*.28` | 付け根と合わせて胴外へ | `Math.min(h*.14,w*.13)` × `Math.min(h*.28,w*.26)` |
| しっぽ | 太さ `h*.2`=11.6・到達 `h*.3` | 胴幅の23%の太さ | `Math.min(h*.2,w*.17)`・到達もw基準 |

**教訓: キャラのパーツ寸法を「高さ基準」だけで書くと、幅を変えた瞬間に破綻する。
片方の軸だけで書かれた寸法は `Math.min(h*a, w*b)` に置き換える。**
今回は1回目のレンダリングで実際に画像を見なければ気づけなかった（数値だけ見ればw=50は達成済みに見えた）。

### 学び2: `preload="auto"` の音声は、ハーネスで「リクエスト失敗2件」として出る

m4aを `preload="auto"` にした初回のハーネスが `requestfailed` 2件でFAILした。
原因はファイルの不在でもMIMEでもなく、**メディア要素がバッファ充足後にリクエストを中断する**ため。
`preload="none"` にしたら0件になり、同時に初期ロードが 2.76MB → 0.03MB になった。
BGMはユーザー操作起点でしか鳴らないので、`preload="none"` に機能上のデメリットがない。

### 学び3: ハーネスの「タップ」チェックは **DOM順で最初の操作要素** を叩く

`page.locator('button, [role="button"], canvas').first()` なので、DOM先頭の要素が
**タイトル画面のオーバーレイに覆われていると、永遠にタップできずタイムアウトする**。

- 元の並び: `#cv`（先頭・可視だがオーバーレイに覆われている）→ **FAIL**
- 直した並び: `#mute`（可視・被りなし）→ `#cv` → `#hud`(`#nextc`) → オーバーレイ群 → **PASS**

**注意した落とし穴**: 最初 `#cv` をDOM末尾へ移したら、`document.querySelector('canvas')` が
HUDの `#nextc`（144×104・hidden）を拾ってしまい、Canvas欄の証跡が別物になった。
**ゲーム本体のcanvasは必ず最初のcanvasのまま**にして、その前に可視ボタンを置くのが正解。

### 学び4: HE-AAC化は「必須寄りの推奨」どころか、ほぼ無条件でやる価値がある

192kbps MP3 2本（8.66MB）→ HE-AAC 64kbps m4a（2.87MB）。**67%削減**でフォルダは17MB→2.8MB。
ffmpegの `-profile:a aac_he` は `aac_at` で通らないので、wav経由の `afconvert -d aach` を使う。

```bash
ffmpeg -y -i in.mp3 -acodec pcm_s16le -ar 44100 -ac 2 /tmp/x.wav
afconvert -f m4af -d aach -b 64000 /tmp/x.wav out.m4a
```

### 検証の証跡

- ハーネス: `docs/harness-reports/266-nekotsumi-2026-09-26T23-28-27-184Z.md` — **8/8 PASS**
- BGM実測（Playwright / Chromium・iPhone相当ビューポート）:
  - プレイ中 `Kina_Takes_the_Lead.m4a` paused=false / volume=0.34
  - ミュートON → `tg.266.mute='1'` ・ label「音：オフ」・`bgm.muted=true`
  - ミュートOFF → `tg.266.mute='0'` ・ volume 0.34 へ復帰
  - ゲームオーバー → プレイ中BGM停止・`Queen_of_the_Living_Room.m4a` 再生開始
  - pageerror / console error **0件**
- いぬの描画: 実レンダリング画像で w=50（ねこ88との並びで確認）・丸胴・耳としっぽが胴内に収まることを目視確認

### 未検証

- **iPhone実機**での再生確認（BGMのフェードイン・ミュート復帰は Chromium 上での実測のみ）
- ハーネスの **iOS 75/25 レイアウト6項目は今回は走っていない**。`#game-shell` / `#control-deck` 構造を持たない
  全画面1タップゲームのため `supported:false` で対象外。操作盤を持つ構成へ作り替えるかは未決
