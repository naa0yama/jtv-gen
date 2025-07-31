# Amatsukaze ロゴ解析システム 仕様調査

## 概要
Amatsukazeのロゴ解析プログラムの動作ロジックと「Insufficient logo frames」エラーの原因分析結果。

## 主要コンポーネント

### 1. ロゴデータ構造
- **LogoData** (AMTLogo.cpp:161-238)
  - YUV各プレーンのA・Bパラメータを保存
  - ロゴファイル(.lgd)の読み書き機能
  - 透明度モデル: Y = A*X + B

### 2. ロゴスキャン・解析
- **LogoScan** (LogoScan.cpp:395-481)
  - 動画フレームからロゴを抽出する中核クラス
  - 背景色検出と統計処理によるロゴ生成

- **LogoAnalyzer** (LogoScan.cpp:647-693)
  - ロゴスキャンの全体制御
  - 2段階処理（初期ロゴ作成→精緻化）

## ロゴ検出アルゴリズム

### 背景色検出ロジック
```cpp
// LogoScan.cpp:187-234
// フレーム周辺部の画素値を収集して単一色背景を判定
for (int x = 0; x < scanw; x++) {
    tmpY.push_back(srcY[x]);  // 上端
    tmpY.push_back(srcY[x + (scanh - 1) * pitchY]);  // 下端
}
// 最小と最大が閾値以上離れている場合、単一色でないと判断
if (abs(tmpY.front() - tmpY.back()) > (thy << (bitdepth - 8))) {
    return false;
}
```

**重要**: 指定した矩形範囲内のみで判定される（画面全域ではない）

### 統計解析による透明度計算
```cpp
// LogoScan.cpp:336-350 - 回帰分析
bool LogoColor::GetAB(float& A, float& B, int data_count) const {
    // 前景色と背景色の関係を回帰直線で近似
    // Y = A * X + B の形でロゴの透明度を表現
    if (std::isnan(A) || std::isnan(B) || std::isinf(A) || std::isinf(B) || A == 0)
        return false;  // ←ここで"Insufficient logo frames"の原因
}
```

### 相関計算による特徴点抽出
```cpp
// LogoScan.cpp:106-222
float CalcCorrelation5x5(const float* k, const float* Y, int x, int y, int w, float* pavg)
```
- 各画素周辺の5x5ウィンドウで特徴量を計算
- AVX/AVX2命令による高速化対応

## GUI設定との対応

### 閾値パラメータ
- GUIの 8,10,12,15 は直接 `thy` パラメータに対応
- 8bit動画の場合: `thy << (8-8) = thy` なので実際の閾値は設定値そのもの

### C API インターフェース
```cpp
// LogoScan.cpp:1190-1203
extern "C" AMATSUKAZE_API int ScanLogo(AMTContext * ctx,
    const tchar * srcpath, int serviceid, const tchar * workfile, const tchar * dstpath,
    int imgx, int imgy, int w, int h, int thy, int numMaxFrames,
    logo::LOGO_ANALYZE_CB cb)
```

## "Insufficient logo frames" エラーの原因

### 1. 統計的不十分性
- **有効フレーム数の不足**: 背景色判定を通過するフレームが少なすぎる
- **回帰分析の失敗**: 前景・背景の関係が数学的に成立しない

### 2. 生成コンテンツ特有の問題
- **フレーム間の変化不足**: 静止画的コンテンツで統計的分散が不足
- **インターレース処理による均一化**: MPEG-2エンコードでノイズが除去されすぎる
- **色の変動不足**: 回帰分析に必要な多様性の欠如

### 3. testsrc2での問題
- **ロゴ位置**: 右上部分は高輝度シアン (R:0, G:255, B:255)
- **コントラスト不足**: 高輝度背景 + 白ロゴ = 判定困難

## 解決策

### 1. 背景パターンの最適化
- **rgbtestsrc**: RGB色空間のテストパターン、適度な多様性
- **mandelbrot**: フラクタルパターン（重い処理でキャッシュ警告あり）
- **gradients**: グラデーションパターン（パラメータ制限あり）

### 2. エンコード設定の調整
```bash
# ノイズを意図的に残す設定
-qcomp 0.8        # より高い値でディテール保持
-qmin 2 -qmax 15  # 品質範囲の調整
-noise_reduction 0 # ノイズ除去を無効化
```

### 3. ロゴ配置の最適化
- **現在**: 右上 `x=w-text_w-40:y=40`
- **推奨**: 背景が暗い部分への移動（左上、中央左など）

## 技術的制約

### FFmpegフィルタの制限
- **mandelbrot**: `duration`パラメータ未対応 → `-t`オプション使用必要
- **gradients**: `nb_frames`パラメータ未対応
- **testsrc/testsrc2**: ロゴ位置での色が解析に不適切

### Amatsukazeの動作条件
- **最小フレーム数**: 統計解析に十分なサンプル数が必要
- **背景多様性**: 前景・背景の関係確立に必要な色度・輝度変化
- **範囲指定**: 手動指定した矩形範囲内のみで解析

## 検証方法

### 1. GUI での確認項目
- **読み込みフレーム数** (`LogoNumRead`)
- **有効フレーム数** (`LogoNumValid`)
- **有効フレーム率** (Valid/Read の比率)

### 2. 成功の指標
- 有効フレーム率 > 10%
- 回帰分析での A ≠ 0, NaN, Inf の条件クリア
- 統計的に十分なサンプル数の確保

## 結論

「Insufficient logo frames」エラーは、生成された映像の統計的均質性が原因。ロゴ解析には：

1. **適度な背景多様性**: フレーム間での輝度・色度変化
2. **統計的十分性**: 回帰分析に必要なサンプル数
3. **前景・背景関係**: 透明度モデル確立に必要な相関

が必要である。JTV-Genの改善により、これらの条件を満たす映像生成が可能。
