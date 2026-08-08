# Changelog

## [Unreleased]

### Changed

- 破壊的変更: `FrameBuilder`と`BitSerializer`を統合し、`AdatFrameSerializer`へ置換。256-bitフレームの二重保持を単一shift registerへ統合し、可変index muxをMSB出力＋shiftへ、bit timingの除算を同一edge列を生成する剰余累積へ変更。旧実装とのcycle単位等価性をequivalence testで検証
- `AdatRx`の`synchronizer_basic`へ`WIDTH: 1`を明示し、1-bit信号の既定8-bit化を解消

## [0.3.0] - 2026-08-07

### Changed

- 破壊的変更: `AdatRx`の公開module境界のオーディオサンプルを`gndless_fixedpoint::FixedPointValue<Q1_23>[8]`へ変更し、`gndless_fixedpoint`を依存packageに追加
- 破壊的変更: `AdatTx`の`channels`入力も`gndless_fixedpoint::FixedPointValue<Q1_23>`境界へ変更
- 破壊的変更: `Smux2Unpacker`の`channels`/`samples`も`gndless_fixedpoint::FixedPointValue<Q1_23>`境界へ変更し、`AdatRx`出力へ直接接続できるようにした。`SAMPLE_WIDTH` parameterを廃止
- 破壊的変更: `Smux2Packer`の`samples`/`channels`も`gndless_fixedpoint::FixedPointValue<Q1_23>`境界へ変更し、`AdatTx`入力へ直接接続できるようにした。`SAMPLE_WIDTH` parameterを廃止
- `AdatRx`→`Smux2Unpacker`の直接接続をloopback testへ追加

## [0.2.0] - 2026-08-02

### Added

- AdatTx→AdatRxの48/44.1kHzレート遷移とS/MUX2停止復帰を検証するpackage内E2E Native Testを追加
- Q8周期referenceと32 interval qualificationによるstream lock、rate transition/欠落時の再qualificationを追加

### Changed

- `AdatRx`のPCM出力を完全frameのatomic commitへ変更し、公開ストローブを`frame_valid`へ改名
- ADATストリームから判別できないS/MUX4を非対応とし、`smux_active`は常にS/MUX2として扱うことを明記

## [0.1.0] - 2026-07-31

- 公開moduleのparam/port doc commentを宣言末尾へ統一し、ADAT説明文の改行を整理
- doc commentの句点と体言止めの表記を整理
- doc commentのsummary、箇条書き、`Examples`見出し、code fence形式を整理
- 各testへ検証目的を示すdoc commentを追加
- 公開型packageをproject名と重複する`Adat`から`Types`へ改名

### Changed

- ADAT RX/TXとS/MUX2を独立packageへ移動
- 内部送信helperを`FrameBuilder`、`BitSerializer`、`NrziEncoder`へ改名
- `adat_rx.veryl`に集中していたunit testを各helperの実装ファイルへ移動
- `AdatRx`は50MHzでのみ実機確認済み、`AdatTx`は実機未確認であることをREADMEとmodule docへ明記
- `AdatRx`のmodule docを公開信号の契約に絞り、NRZI復号後の詳細frame図をREADMEへ移動
- `AdatTx`のWavedromを公開信号の契約へ整理し、S/MUX2 pack/unpackの時間順序図を追加
- `AdatRx`のS/MUX2受信案内を`Smux2Unpacker`へ明確化
