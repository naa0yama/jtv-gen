# TSDuck PSI/SI Table Injection ナレッジ

## 問題の概要

`generate_dummy.sh`でEITテーブル（PID 18）とTOTテーブル（PID 20）の注入が失敗する問題が発生。TSDuckの`inject`プラグインで`--inter-packet`や`--bitrate`オプションを使用しても新しいPIDが作成されない。

## 根本原因

**TSDuckは既存のストリームに存在しないPIDを作成する場合、PAT（Program Association Table）とPMT（Program Map Table）の再構築が必須**

### 技術的詳細

1. **PAT（PID 0）**: 全てのプログラムとそのPMT PIDを定義
2. **PMT（PID 257）**: 特定のプログラムの構成要素（映像、音声、データ）を定義
3. **新PID作成**: EIT（PID 18）、TOT（PID 20）などの新しいPIDを作成するには、これらの基礎テーブルが適切に設定されている必要がある

## 失敗パターン

```bash
# ❌ これは動作しない - 既存PIDのみを部分的に置換
tsp --japan -I file input.ts \
    -P inject sdt.bin --pid 17 --replace \
    -P inject eit.bin --pid 18 --inter-packet 30 \
    -P inject tot.bin --pid 20 --inter-packet 150 \
    -P continuity --fix \
    -O file output.ts
```

**結果**: EIT（PID 18）とTOT（PID 20）は作成されない

## 成功パターン

```bash
# ✅ これは動作する - 包括的PSI/SI再構築
tsp --japan -I file input.ts \
    -P inject pat.bin --pid 0 --replace \
    -P inject cat.bin --pid 1 --inter-packet 300 \
    -P inject nit.bin --pid 16 --inter-packet 100 \
    -P inject sdt.bin --pid 17 --replace \
    -P inject eit.bin --pid 18 --inter-packet 50 \
    -P inject tot.bin --pid 20 --inter-packet 150 \
    -P inject pmt.bin --pid 257 --replace \
    -P continuity --fix \
    -P regulate --bitrate 15000000 \
    -O file output.ts
```

**結果**: 全てのPID（0, 1, 16, 17, 18, 20, 257, 273, 274）が正しく作成される

## 必要なテーブル構造

### PAT (Program Association Table)
```xml
<PAT version="3" current="true" transport_stream_id="XXX" network_PID="16">
  <service service_id="XXX" program_map_PID="257"/>
</PAT>
```

### PMT (Program Map Table)
```xml
<PMT version="12" current="true" service_id="XXX" PCR_PID="273">
  <component elementary_PID="273" stream_type="0x02">
    <stream_identifier_descriptor component_tag="0"/>
  </component>
  <component elementary_PID="274" stream_type="0x0F">
    <stream_identifier_descriptor component_tag="16"/>
  </component>
</PMT>
```

### その他必要なテーブル
- **CAT**: Conditional Access Table（PID 1）
- **NIT**: Network Information Table（PID 16）
- **SDT**: Service Description Table（PID 17）
- **EIT**: Event Information Table（PID 18）
- **TOT**: Time Offset Table（PID 20）

## TSDuckプラグインのオプション

### --replace vs --inter-packet
- `--replace`: 既存のPIDの内容を置換（PIDが既に存在する必要がある）
- `--inter-packet N`: N個のパケット間隔で新しいPIDを注入（新PID作成可能）

### 重要なプラグイン
- `continuity --fix`: パケット連続性カウンターを修正
- `regulate --bitrate N`: 出力ビットレートを調整

## 実装上の注意点

1. **テーブル作成順序**: PAT → CAT → NIT → SDT → EIT → TOT → PMT
2. **TSDuck注入順序**: 基礎テーブル（PAT, PMT）を先に処理
3. **ARIB準拠**: 日本の放送規格に準拠するため`--japan`オプション必須
4. **ビットレート調整**: `regulate`プラグインで全体のビットレート制御

## generate_standalone_ts.sh との違い

| 項目 | generate_dummy.sh（修正前） | generate_standalone_ts.sh |
|------|---------------------------|---------------------------|
| PSI/SI範囲 | 部分的（SDT, EIT, TOT） | 包括的（PAT, CAT, NIT, SDT, EIT, TOT, PMT） |
| PAT更新 | なし | あり |
| PMT更新 | なし | あり |
| 新PID作成 | 失敗 | 成功 |

## 解決策

`generate_dummy.sh`を`generate_standalone_ts.sh`と同じアプローチに変更：

1. **包括的テーブル生成**: PAT, CAT, PMT, NIT, SDT, EIT, TDT, TOT
2. **適切な注入順序**: 基礎テーブルから順次処理
3. **regulate プラグイン追加**: ビットレート制御
4. **完全なPSI/SI再構築**: 部分的更新ではなく全面的再構築

この知見により、TSDuckでの新PID作成には包括的なPSI/SI再構築が必須であることが判明した。