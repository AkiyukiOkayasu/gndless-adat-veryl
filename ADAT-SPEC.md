# ADAT Optical Interface 仕様メモ

ADAT Optical Interface (通称 ADAT Lightpipe) の基本仕様を、実装とは独立してまとめたメモです。

## 概要

- 1990年にAlesisが設計したマルチチャンネルデジタルオーディオ伝送技術
  - ADATはもともとは8chのMTRの製品名。この伝送技術がADAT Optical Interfaceとして現在も様々なオーディオインターフェースなどで使用されている
- 1本のTOSLINK光ファイバで **8ch × 24bit × 44.1/48kHz** を伝送する
- S/MUX(サンプルレート拡張):
  - **S/MUX2** (88.2/96kHz): 4論理ch — Alesis 2001年addendumで正式定義
  - **S/MUX4** (176.4/192kHz): 2論理ch — **公式仕様は存在しない（未定義）**
- 本フォーマットはチャンネルステータスやサンプルレート情報を一切持たない（S/PDIFやAES3と違い、受信側がレートを自動判別する手段がない）

## 物理・電気仕様

| 項目 | 値 |
| --- | --- |
| コネクタ | TOSLINK |
| ケーブル長 | 実用的には5m以内 |
| 物理ビットレート (48kHz系) | **12.288 Mbps** (256bit × 48000/s) |
| 物理ビットレート (44.1kHz系) | **11.2896 Mbps** (256bit × 44100/s) |
| フレームレート | 44.1/48kHz |
| 変調 | NRZI（遷移 = 1、無遷移 = 0） |
| クロック | 受信側で信号から回復する（外部クロック不要） |

- ビットレートは44.1kHz系と48kHz系の2種類だけで、**S/MUX2/S/MUX4でも物理ビットレート・フレームレートは変化しない**。
- 物理層はS/PDIFと同じ

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

## UserBit

- 各フレームに4bitのuser bit領域がある（伝送位置11–14）。
- 歴史的にはSMPTEタイムコードやMIDIの転送に使うことが想定されていたが、
  **現代の実機では実質的にほぼ未使用**であり、多くの機器は常に0を送る。
- 事実上唯一の標準的な使用が **S/MUX2フラグ**:
  - Alesis 2001年2月のADDENDUM "2X Sample Rate (96kHz) Specification" で定義
  - **送信順で2番目のuser bit (U2) がS/MUX2時のみ1になる**
  - 88.2/96kHzのときだけ1。44.1/48kHzでは0。
  - **176.8/192kHzのときも0になることに注意**。U2はS/MUX2専用フラグである。
- S/PDIFやAES3のような**チャンネルステータスやサンプルレート情報は存在しない**（このためレートの判別が問題になる）。

## S/MUX (Sample Multiplexing)

S/MUXは物理レイヤを変えず、フレーム内のスロット割り当てだけで88.2/96kHzの2倍レート、176.4/192kHzの4倍レートを実現する。

- **物理ビットレート・フレームレートは通常時と同一**（48kHz系は常に12.288 Mbps / 48000 frames/s）。
  したがってストリームのタイミングからは通常8chとS/MUX2, S/MUX4は判別できない。
- サンプル数: 8スロット × 24bit × 48000/s = 9.216 Mbpsのペイロード容量を、論理ch数とレートの積で使い分ける。

### S/MUX2 (88.2/96kHz): 4論理ch

| ADAT物理スロット | 論理チャンネル |
| --- | --- |
| 0, 1 | 論理ch 0 |
| 2, 3 | 論理ch 1 |
| 4, 5 | 論理ch 2 |
| 6, 7 | 論理ch 3 |

- 各論理chは2物理スロットを使い、サンプルを交互に分散して96kHz（または88.2kHz）を得る。
- 2サンプルを1フレームに格納するため、原理的に1サンプルの遅延が発生する。
- **UserBit U2（送信順2番目）= 1** がS/MUX2のフラグ。受信側はこれで検出できる。

### S/MUX4 (176.4/192kHz): 2論理ch

**注意: S/MUX4に公式仕様は存在しない（未定義）。**
メーカーごとによる実装に差がある可能性もあるため、**S/MUX4は使うべきではない**。176.8kHz以上を伝送するならAES3等を用いるべきである。

| ADAT物理スロット | 論理チャンネル (RME Quad Wire) |
| --- | --- |
| 0, 1, 2, 3 | 論理ch 0 |
| 4, 5, 6, 7 | 論理ch 1 |

- RMEの実装では各論理chは4物理スロットを使い、192kHz（または176.4kHz）を得る。
- 4サンプルを1フレームに格納するため、原理的に3サンプルの遅延が発生する。
- **UserBitフラグは無い**（全bit 0になる）。ストリームからの自動判別は不可能。受信側はユーザーによるクロック/レートの手動設定が必要。

## 公式・参考文献

- Alesis "ADAT Proprietary Multichannel Optical Digital Interface" (1990, 基本仕様)
  - https://datasheet.datasheetarchive.com/originals/crawler/wavefrontsemi.com/70bb075285d9015650136d4e2cc6084e.pdf
- Alesis "ADDENDUM February 2001 2X Sample Rate (96kHz) Specification" (S/MUX2の定義、UserBit U2)
  - 原文は配布が限定的。内容はxmos lib_adatのIssue #6で引用されている
  - https://github.com/xmos/lib_adat/issues/6
- Wavefront Application Note AN3101-09 "S-Mux Transmitter for ADAT Optical Protocol"
  - Wavefront SemiconductorはAlesisの半導体部門が分社してできた
  - https://freeverb3-vst.sourceforge.io/doc/wavefrontsemi/WavefrontAN3101-09%20s-Mux%20Transmitter%20for%20ADAT%20Optical%20Protocol.pdf
- Wavefront Application Note AN3101-10 "S-Mux Receiver for ADAT Optical Protocol"
  - https://freeverb3-vst.sourceforge.io/doc/wavefrontsemi/WavefrontAN3101-10%20S-Mux%20Receiver%20for%20ADAT%20Optical%20Protocol.pdf
- Wavefront AL1401 (ADATエンコーダ) / AL1402 (ADATデコーダ) データシート
  - https://www.cabintechglobal.com/pdf/WAVEFRONT_AL1401.pdf
  - 現行互換品: CoolAudio V1401 / V1402
- Benchmark ADC16 "No S/MUX4 Flag" (S/MUX4にフラグが無いことがマニュアルに明記されている)
  - https://www.bhphotovideo.com/lit_files/732209.pdf
