# PC-8801 (YM2203[OPN] / YM2608[OPNA]) 高精度ポルタメントルーチン (Z80)

昔PC-8801シリーズのFM音源ドライバ向けに制作した、Z80アセンブリによるポルタメント（ピッチベンド）制御ルーチンです。
アルゴリズムの公開を目的として、可読性向上のための修正と、ドライバ依存部の外部スタブ化を行った汎用サンプル版を公開しています。

また、比較資料として、実際にドライバへ組み込まれていたオリジナル版アセンブルリスト（マシン語ダンプ付き）も収録しています。オリジナル版では、ビットシフト幅を実用範囲（約3.5オクターブ）に絞った `MODE11` を実装しており、全8オクターブ対応の公開版（`MODE13`）よりもさらなる高速化を図っています。

---

## 技術的特徴・アルゴリズム

当時のサウンドドライバで一般的だった「F-Numberへの単純加算」や「大容量のピッチテーブル参照」ではなく、以下の手法を組み合わせることで**省メモリ・高速処理**と**高精度な音程遷移**を両立しています。

### 1. 実測キャリブレーションによるBlock閾値とヒステリシス制御
YM2203 / YM2608特有のオクターブ（Block）跨ぎにおける周波数の不連続性を解消するため、実測値から算出した閾値でBlock遷移を制御しています。また、上昇時と下降時で折り返し地点に幅（ヒステリシス）を持たせることで、境界付近での音程のバタつきを防いでいます。

* **上昇時（Block + 1）：** F-Number が `1236` 以上で Block を1つ上げ、折り返し基準値 `618` へ遷移（`618 × 2 = 1236`）
* **下降時（Block - 1）：** F-Number が `584` 未満で Block を1つ下げ、折り返し基準値 `1168` へ遷移（`584 × 2 = 1168`）

> **開発メモ：** これらの閾値は、開発当時、詳細なデータシートが手元にない環境でPC-8801実機での実測キャリブレーションのみで探り当てた数値です。後年YM2203の仕様と照らし合わせたところ、`618` は「C（ド）」、`1168` は「B（シ）」のF-Numberとほぼ一致しており、実測で導き出した境界値が「1オクターブ＝周波数2倍」の物理法則およびチャタリング防止のヒステリシス制御と合致していました。

### 2. 誤差累積（剰余繰り越し）による「0〜+1」の着地精度
除算命令を持たないZ80上で、1tickごとの除算で生じた「余り（剰余）」を次回の分子へ加算する（ブレゼンハムアルゴリズムの応用）ことで、割り切れない音長や移動幅であっても、指定ステップ完了時に目標周波数へ **「0〜+1」の誤差** で一致します。

### 3. 音長と移動幅に応じた3段階の除算切り替え（負荷平準化）
四分音符＝48tick（全音符＝192tick）の時間軸を前提とし、音長（ステップ数）と総変化量に応じて3つの除算ロジックを動的に切り替えることで、タイマー割り込み内の実行ステート数を平準化しています。

| モード | 適用条件 | 精度 / 方式 | 実測ステート数 | 設計意図 |
| :--- | :--- | :--- | :--- | :--- |
| **`MODEC`** | ステップ数 ≥ 128 | 減算カウンタループ | **約100〜300** *(※)* | 全音符（192tick）等のロングポルタメント想定。1tickあたりの商が小さいため減算ループが最速となる。 |
| **`MODE8`** | ステップ数 < 128 かつ 分子 < 256 | 8bit シフト除算 | **約400** | 変化量が小さい場合の中速・省クロック処理。 |
| **`MODE13`** | ステップ数 < 128 かつ 分子 ≥ 256 | 13bit シフト除算 | **約580** | 短い音符での大跳躍に対応。全8オクターブをオーバーフローなしでカバー（オリジナル版は11bit幅の `MODE11`：約550ステート）。 |

*(※) 全音符（192tick）で全8オクターブ横断（変化量 約4800）を行う極限ケースでも `4800 ÷ 192 = 25ループ`（約725ステート）に収まります。*

---

## 他方式との比較

当時のZ80環境でよく用いられた他のポルタメント実装方式との比較です。

| 方式 | 処理速度（1tickあたり） | メモリ消費量 | 特徴・ゲーム組み込み時の課題 |
| :--- | :--- | :--- | :--- |
| **大規模テーブル引き** | 最速（数十ステート） | **膨大（数KB〜10KB超）** | 速度・精度ともに優秀だがメモリ消費が大きく、ゲームではグラフィック等の領域を圧迫する。 |
| **固定小数点加算**<br>*(16.8bit / 16.16bit)* | 中速（約350〜450ステート） | 小（除算・ワーク込みで約250〜350バイト） | 発音開始時の除算スパイクや、ロングポルタメント時の丸め誤差蓄積による着地ズレが起きやすい。 |
| **本ルーチン**<br>*(ブレゼンハム＋3段階除算)* | **高速〜中速**<br>（**約100〜580ステート**） | **小（テーブル不要・310バイト）** | **初期化の除算ゼロ**・**「0〜+1」の着地精度**・**Block跨ぎヒステリシス制御**を省メモリで両立。 |

---

## 初期化ルーチン（`SUB25`）の総変化量算出式

開始音階と終了音階のBlockおよびF-Numberから、目標までの総変化量 $F$ を以下の3パターンで算出します。

1. **同一オクターブ内の場合（`S25A`）**
   * 上昇時：$F = \text{終了FNum} - \text{開始FNum}$
   * 下降時：$F = \text{開始FNum} - \text{終了FNum}$
2. **下降・オクターブ跨ぎの場合（`S25B`）**
   $$F = (\text{開始FNum} - 584) + (1168 - \text{終了FNum}) + 584 \times (\text{オクターブ差} - 1)$$
3. **上昇・オクターブ跨ぎの場合（`S25C`）**
   $$F = (1236 - \text{開始FNum}) + (\text{終了FNum} - 618) + 618 \times (\text{オクターブ差} - 1)$$

---

## ファイル構成

* `src/portamento.z80` : 汎用サンプル版ソースコード（`SUB25` 初期化ルーチン ＋ `PORTA` メインルーチン）
* `original/portamento_orig.txt` : ドライバ組込済のオリジナル版アセンブルリスト（マシン語ダンプ付き・`MODE11` 実装）

---

## 組み込み仕様（外部スタブ）

`src/portamento.z80` はアルゴリズム単体のサンプルコードであるため、単体では動作しません。組み込むサウンドドライバ側で以下の3つの外部ルーチンを用意してください。

* **`FNUM`** : 音階番号（`A = 1〜96`）に対応するYM2203/YM2608用の `Block + F-Number`（`HL`）を返すルーチン
* **`FMON`** : 初期周波数（`HL`）のレジスタ設定およびキーオンを行うルーチン
* **`FMOUT`** : ポルタメント経過中の周波数レジスタ（`0A4H` / `0A0H`）へ新しい値（`DE`）を書き込むルーチン

---

## 参考文献・謝辞 (References)

当時のFM音源レジスタ書き込み処理（`FMOUT`）の実装およびハードウェア制御にあたり、以下の書籍を参考にさせていただきました。

* 日高 徹 著『PC-8801シリーズ マシン語サウンドプログラミング』（株式会社アスキー）

---

## ライセンス

MIT License

## Overview (English)

**High-Precision, Low-Memory Portamento (Pitch Bend) Routine for Z80 / YM2203 (OPN) & YM2608 (OPNA)**

Originally developed for NEC PC-8801 sound drivers, this Z80 assembly routine achieves smooth, accurate pitch transitions without relying on large lookup tables or fixed-point arithmetic.

### Key Features
* **Bresenham-Style Error Accumulation:** Carries over division remainders per tick, guaranteeing a landing accuracy within **0 to +1** of the target F-Number even on non-divisible note lengths.
* **Hysteresis Block Switching:** Prevents pitch chattering across octave (Block) boundaries using empirically calibrated thresholds (`1236` -> `618` upward, `584` -> `1168` downward).
* **3-Stage Adaptive Division:** Dynamically switches between subtraction loop (`MODEC`), 8-bit shift (`MODE8`), and 13-bit shift (`MODE13`) based on step length and delta to balance CPU load (**~100–580 states/tick**).
* **Compact Footprint:** Requires **zero lookup tables** and uses only **310 bytes** of code.

*(Note: Includes both the refactored generic sample with external driver stubs and the original hardware-tested assembly dump featuring the faster 11-bit `MODE11`.)*

