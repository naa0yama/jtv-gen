# Amatsukaze ロゴスキャン解析ドキュメント

## 概要

Amatsukazeのロゴスキャン機能は、日本のテレビ放送に含まれるロゴ（放送局ロゴなど）を自動検出・解析するシステムです。このドキュメントでは、ロゴスキャンアルゴリズムの詳細と「Insufficient logo frames」エラーの原因・対策について解説します。

## アーキテクチャ概要

### 主要クラス構成

```
LogoAnalyzer (AMTObject)
├── InitialLogoCreator (SimpleVideoReader)
│   └── LogoScan
│       └── LogoColor[] (Y, U, V)
└── LogoDataParam (LogoData)
    ├── mask[]
    ├── kernels[]
    └── scales[]
```

## ロゴスキャンアルゴリズム

### 1. フレーム検証プロセス (`LogoScan::AddFrame`)

#### 1.1 背景色計算
```cpp
// 画面境界のピクセル値を収集
for (int x = 0; x < scanw; x++) {
    tmpY.push_back(srcY[x]);                           // 上境界
    tmpY.push_back(srcY[x + (scanh - 1) * pitchY]);   // 下境界
}
for (int y = 1; y < scanh - 1; y++) {
    tmpY.push_back(srcY[y * pitchY]);                  // 左境界
    tmpY.push_back(srcY[scanw - 1 + y * pitchY]);     // 右境界
}
```

#### 1.2 背景色均一性チェック
```cpp
std::sort(tmpY.begin(), tmpY.end());
if (abs(tmpY.front() - tmpY.back()) > (thy << (bitdepth - 8))) {
    return false;  // フレーム無効
}
```

**検証条件:**
- Y成分の変動が `thy << (bitdepth - 8)` 以下
- U、V成分も同様の条件を満たす
- すべての条件をクリアした場合のみ有効フレーム

#### 1.3 パラメータ詳細
- **thy**: 背景色均一性の閾値（デフォルト: 30*8=240、8bitでは12程度）
- **bitdepth**: ピクセルビット深度（8bit/10bit対応）
- **scanw/scanh**: スキャン領域のサイズ

### 2. ロゴデータ生成プロセス (`LogoScan::GetLogo`)

#### 2.1 回帰直線分析 (`LogoColor::GetAB`)
```cpp
bool LogoColor::GetAB(float& A, float& B, int data_count) const {
    // 通常方向の回帰直線
    approxim_line(data_count, sumF, sumB, sumF2, sumFB, A1, B1);
    // XY入れ替えの回帰直線
    approxim_line(data_count, sumB, sumF, sumB2, sumFB, A2, B2);
    
    // 両方の結果を平均化
    A = (float)((A1 + (1 / A2)) / 2);
    B = (float)((B1 + (-B2 / A2)) / 2);
    
    // 無効値チェック
    if (std::isnan(A) || std::isnan(B) || std::isinf(A) || std::isinf(B) || A == 0)
        return false;
    
    return true;
}
```

#### 2.2 ロゴデータ生成
```cpp
for (int y = 0; y < scanh; y++) {
    for (int x = 0; x < scanw; x++) {
        int off = x + y * scanw;
        if (!logoY[off].GetAB(aY[off], bY[off], nframes)) 
            return nullptr;  // ← "Insufficient logo frames"エラー
    }
}
```

## エラー解析: "Insufficient logo frames"

### エラー発生箇所

1. **初期ロゴ作成時** (`InitialLogoCreator::readAll`, Line 872)
2. **ロゴ再生成時** (`LogoAnalyzer::ReMakeLogo`, Line 450)

### エラー原因詳細

#### 1. 背景の不安定性
- **動きのある背景**: スポーツ中継、ニュース映像
- **グラデーション背景**: CG背景、アニメーション
- **ノイズ**: エンコード時の圧縮ノイズ

#### 2. ロゴ周辺の複雑さ
- **エッジ効果**: ロゴ境界のアンチエイリアシング
- **色合成**: 半透明ロゴの色混合
- **インターレース**: フィールド間の色差

#### 3. パラメータ設定
- **thy値が小さすぎる**: 厳しすぎる背景色判定
- **スキャン領域**: 不適切な領域設定
- **フレーム数不足**: 十分なサンプルフレームが収集できない

### 具体的な失敗ケース

```cpp
// Case 1: 回帰直線の傾きが0
A = 0  // ロゴの透明度情報が取得できない

// Case 2: 数値計算エラー
A = NaN, B = NaN  // 統計的に無効なデータ

// Case 3: 無限値
A = ∞, B = ∞  // 分母が0になる計算
```

## 対策と改善方法

### 1. 映像品質の改善

#### JTV-Gen側の対策
```bash
# より安定したロゴパターンの生成
drawtext=fontsize=36:fontcolor=white@0.8:text='JTV-Gen':x=w-text_w-40:y=40:box=1:boxcolor=black@0.8:boxborderw=5
```

**改善点:**
- 一貫した背景色（black@0.8のボックス）
- 適切なロゴサイズ（36px）
- 固定位置（右上コーナー）

#### 背景パターンの最適化
```bash
# テストパターンの安定化
testsrc2=size=1440x1080:rate=30000/1001:duration=$duration
```

### 2. パラメータ調整

#### thy値の調整案
```cpp
// 現在: thy = 30 (8bit時は12程度)
// 推奨: thy = 50-100 (より寛容な判定)
```

#### スキャン領域の最適化
```cpp
// ロゴ周辺の安定した領域を選択
scanx = logo_x - margin;
scany = logo_y - margin;
scanw = logo_width + margin * 2;
scanh = logo_height + margin * 2;
```

### 3. 前処理の実装

#### デノイズ処理
```bash
# FFmpegでのノイズ除去
-vf "hqdn3d=luma_spatial=4:chroma_spatial=3"
```

#### インターレース処理
```bash
# インターレース解除
-vf "yadif=mode=1:parity=auto:deint=interlaced"
```

## 実装上の注意点

### 1. メモリ管理
- `LogoScanDataCompressed`: 大量フレームデータの圧縮保存
- `std::unique_ptr`: RAII による自動メモリ管理

### 2. パフォーマンス最適化
- **SIMD命令**: AVX/AVX2 による高速相関計算
- **並列処理**: マルチスレッドでのフレーム解析
- **メモリアラインメント**: キャッシュ効率の向上

### 3. 精度向上
- **ビット深度対応**: 8bit/10bit/16bit
- **色空間**: YUV420p/YUV422p/YUV444p
- **フィールド処理**: インターレース映像対応

## デバッグ情報

### ログ出力例
```
maxi = 8 (67.5%)  // 最適fade値とその確率
1000 frames       // 処理フレーム数
LogoScanH450 Message: Insufficient logo frames  // エラーメッセージ
```

### 診断手順
1. **フレーム受け入れ率** (`numFrames / totalFrames`)
2. **背景色変動値** (`abs(tmpY.front() - tmpY.back())`)
3. **thy閾値との比較** (`variation <= (thy << (bitdepth - 8))`)
4. **回帰分析結果** (`A != 0, !isnan(A), !isnan(B)`)

## 関連ファイル

- `LogoScan.h`: クラス定義とインターフェース
- `LogoScan.cpp`: 主要アルゴリズム実装
- `AMTLogo.h`: ロゴデータ構造体
- `ComputeKernel.cpp`: SIMD最適化関数

## 参考情報

- **透過性ロゴフィルタプラグイン**: MakKi氏による元実装
- **MIT License**: Nekopanda氏によるAmatsukaze実装
- **GitHub**: https://github.com/makiuchi-d/delogo-aviutl

---

*このドキュメントは Amatsukaze v0.x.x のソースコード解析に基づいています。*