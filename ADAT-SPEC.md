# ADAT Optical Interface 仕様メモ

ADAT Optical Interface (通称 ADAT Lightpipe) の基本仕様を、実装とは独立してまとめたメモです。
情報が散逸しやすいため、公式資料の内容と実機観測を整理してここに集約します。

## 概要

- 1990年にAlesisが設計したマルチチャンネル光デジタルオーディオインターフェース
- 1本のTOSLINK光ファイバで **8ch × 24bit × 44.1/48kHz** を伝送する
- サンプルレート拡張フォーマット:
  - **S/MUX2** (88.2/96kHz): 4論理ch — Alesis 2001年addendumで正式定義
  - **S/MUX4** (176.4/192kHz): 2論理ch — **公式仕様は存在しない（未定義）**。RMEのQuad Wire等のデファクト実装
- ADAT IPはWavefront Semiconductorがライセンス管理している（2011年にAlesisの半導体部門として分社）
- 本フォーマットはチャンネルステータスやサンプルレート情報を一切持たない（S/PDIFやAES3と違い、受信側がレートを自動判別する手段がない）

## 物理・電気仕様

| 項目 | 値 |
| --- | --- |
| コネクタ | TOSLINK (JIS F05 / EIAJ RC-5720B) |
| 光ファイバ | 1mm POF (プラスチック光ファイバ) |
| 光源 | 660nm 赤色LED |
| ケーブル長 | 実用的には5m以内（TOSLINK規格の技術的最大は10m程度） |
| ビットレート (48kHz系) | **12.288 Mbps** (256bit × 48000/s) |
| ビットレート (44.1kHz系) | **11.2896 Mbps** (256bit × 44100/s) |
| フレームレート | サンプルレートと等しい 44.1/48kHz |
| 変調 | NRZI（遷移 = 1、無遷移 = 0） |
| クロック | 受信側で信号から回復する（外部クロック不要） |

- ビットレートは44.1kHz系と48kHz系の2種類だけで、**S/MUX2/S/MUX4でも物理ビットレート・フレームレートは変化しない**。
  倍レート化は「1フレームに運ぶサンプルを複数スロットへ分散する」ことによって実現する（後述）。

## フレーム構造 (256bit)

```text
伝送位置    0          10 11       15 16                  45             226                255
             |           | |         | |                    |               |                  |
伝送順  →  +-------------+-----------+----------------------+----- ... -----+------------------+
            | SYNC        | User      | CH0                  |               | CH7              |
            | 10bit + sep | 4bit+sep  | 6 * (4bit + sep)    |    CH1-CH6    | 6*(4bit + sep)  |
           +-------------+-----------+----------------------+----- ... -----+------------------+
bit数           11             5           30                    180             30
```

| 伝送位置 | 内容 |
| ---: | --- |
| 0–9 | SYNCの0を10bit（意図的な無遷移区間 = フレーム境界） |
| 10 | SYNC separator = 1 |
| 11–14 | User bit 4bit（送信順: U3が先頭、U0が最後） |
| 15 | User separator = 1 |
| 16–255 | CH0–CH7、各30bit |

- 1フレーム = 256bit = 11 + 5 + 8×30
- 各チャンネルは24bit PCMをMSB-firstで6個のニブルに分け、各4bitの後ろにseparator = 1を置く（30bitエンコード）:

```text
channel内の伝送順 →
+-------------+-------------+-------------+-------------+-------------+-------------+
| D23..D20  1 | D19..D16  1 | D15..D12  1 | D11..D8   1 | D7..D4    1 | D3..D0    1 |
+-------------+-------------+-------------+-------------+-------------+-------------+
    nibble 0      nibble 1      nibble 2      nibble 3      nibble 4      nibble 5
```

- データ部は最大5bit連続（4データ + 1 sync separator）で、NRZI上は必ず5bitごとに遷移が生じる。
  これがクロック回復のための同期機会になる。
- 逆にSYNCの10bit連続0は、データ部では決して現れない無遷移区間であり、フレーム境界の検出に使われる。
- **チャンネルステータスやサンプルレート情報は存在しない**（このため倍レートの判別が問題になる）。

## UserBit

- 各フレームに4bitのユーザーデータ領域がある（伝送位置11–14）。
- 歴史的にはSMPTEタイムコードやMIDIの転送に使うことが想定されていたが、
  **現代の実機では実質的にほぼ未使用**であり、多くの機器は常に0を送る。
- 事実上唯一の標準的な使用が **S/MUX2フラグ**:
  - Alesis 2001年2月のADDENDUM "2X Sample Rate (96kHz) Specification" で定義
  - **送信順で2番目のuser bit (U2) がS/MUX2時のみ1になる**
  - 88.2/96kHzのときだけ1。44.1/48kHzでは0
- **S/MUX4 (176.4/192kHz) ではuser bitは全て0**（フラグを持たない）。
  RME ADI-8 DSマニュアルの "No S/MUX4 Flag" に明記されている。S/MUX4の受信は
  ユーザーによるクロック/レート設定で行う。
- メーカーによる実装のばらつき: RME Babyface ProはS/MUX2時のみU2 = 1、
  S/MUX4時は全bit 0（ロジアナ実測で確認済み）。フラグを全く立てない機器もある。

## S/MUX (Sample Multiplexing)

S/MUXは物理レイヤを変えず、フレーム内のスロット割り当てだけで倍レートを実現する。

- **物理ビットレート・フレームレートは通常時と同一**（48kHz系は常に12.288 Mbps / 48000 frames/s）。
  したがってストリームのタイミングだけからは通常8chとS/MUX2は判別できない。
- サンプル数: 8スロット × 24bit × 48000/s = 9.216 Mbpsのペイロード容量を、論理ch数と
  レートの積で使い分ける。

### S/MUX2 (88.2/96kHz): 4論理ch

| ADAT物理スロット | 論理チャンネル |
| --- | --- |
| 0, 1 | 論理ch 0 |
| 2, 3 | 論理ch 1 |
| 4, 5 | 論理ch 2 |
| 6, 7 | 論理ch 3 |

- 各論理chは2物理スロットを使い、サンプルを交互に分散して96kHz（または88.2kHz）を得る。
- **UserBit U2（送信順2番目）= 1** がS/MUX2のマーカー。受信側はこれで検出できる。

### S/MUX4 (176.4/192kHz): 2論理ch

> **注意: S/MUX4に公式仕様は存在しない（未定義）。** 以下はRMEのQuad Wire方式という
> デファクト実装の説明であり、メーカーによってスロット割り当てが異なる可能性がある。

| ADAT物理スロット | 論理チャンネル (RME Quad Wire) |
| --- | --- |
| 0, 1, 2, 3 | 論理ch 0 |
| 4, 5, 6, 7 | 論理ch 1 |

- RMEの実装では各論理chは4物理スロットを使い、192kHz（または176.4kHz）を得る。
- **UserBitフラグは無い**（全bit 0）。ストリームからは判別できないため、
  受信側はユーザーのクロック/レート設定に依存する。

## 公式・参考文献

- Alesis "ADAT Proprietary Multichannel Optical Digital Interface" (1990, 基本仕様)
  - https://datasheet.datasheetarchive.com/originals/crawler/wavefrontsemi.com/70bb075285d9015650136d4e2cc6084e.pdf
- Alesis "ADDENDUM February 2001 2X Sample Rate (96kHz) Specification" (S/MUX2の定義、UserBit U2)
  - 原文は配布が限定的。内容はxmos lib_adatのIssue #6で引用されている
  - https://github.com/xmos/lib_adat/issues/6
- Wavefront Application Note AN3101-09 "S-Mux Transmitter for ADAT Optical Protocol"
  - https://freeverb3-vst.sourceforge.io/doc/wavefrontsemi/WavefrontAN3101-09%20s-Mux%20Transmitter%20for%20ADAT%20Optical%20Protocol.pdf
- Wavefront Application Note AN3101-10 "S-Mux Receiver for ADAT Optical Protocol"
  - https://freeverb3-vst.sourceforge.io/doc/wavefrontsemi/WavefrontAN3101-10%20S-Mux%20Receiver%20for%20ADAT%20Optical%20Protocol.pdf
- Wavefront AL1401 (ADATエンコーダ) / AL1402 (ADATデコーダ) データシート
  - https://www.cabintechglobal.com/pdf/WAVEFRONT_AL1401.pdf
  - 現行互換品: CoolAudio V1401 / V1402
- RME ADI-8 DSマニュアル "No S/MUX4 Flag" (S/MUX4にフラグが無いことの明記)
  - https://www.bhphotovideo.com/lit_files/732209.pdf
- RME HDSPe AIO Pro仕様 ("Alesis specification"と謳うS/MUX 4ch@96kHz / S/MUX4 2ch@192kHz。
  S/MUX4部分は公式仕様が存在しないためデファクト実装としての記述)
  - https://rme-audio.de/hdspe-aio-pro.html
- RME ADI-2 FSマニュアル (S/MUXとS/MUX4のチャンネルマッピング)
  - https://rme-audio.de/downloads/adi2fs_e.pdf
- xmos lib_adat (実装リファレンス、S/MUXの並べ替えは上位レイヤで行う方針)
  - https://github.com/xmos/lib_adat
- Wikipedia "ADAT Lightpipe" (概要)
  - https://en.wikipedia.org/wiki/ADAT_Lightpipe

## 実機観測メモ (RME Babyface Pro + Saleae Logic)

- 96kHz S/MUX2 (無音): フレーム周期20.83µs (48k)、ビットレート12.288 Mbps、
  user bit = [0, 1, 0, 0]（U2 = 1）— 物理レート不変・U2フラグを実測確認
- 176.4/192kHz S/MUX4: user bit = 全0 — フラグなしを実測確認
