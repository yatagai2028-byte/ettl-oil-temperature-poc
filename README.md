# ETTデータを用いた変圧器オイル温度予測 PoC

## 概要

本リポジトリは、ETT（Electricity Transformer Temperature）データセットを用いて、
変圧器のオイル温度 `OT` を将来予測し、その予測が運用・予防保全判断にどの程度活用できるかを検証したPoCです。

単に予測誤差を小さくするだけでなく、

- どの情報が予測に有効か
- どのモデルが妥当か
- 何時間先まで予測価値があるか
- 実運用に使う際の限界は何か

まで含めて検証しています。

## 使用データ

ETT-small の4ファイルを使用しています。

| ファイル | 対象 | 時間粒度 |
|---|---|---|
| ETTh1.csv | Transformer 1 | 1時間 |
| ETTh2.csv | Transformer 2 | 1時間 |
| ETTm1.csv | Transformer 1 | 15分 |
| ETTm2.csv | Transformer 2 | 15分 |

主な列は以下です。

- `date`: 時刻
- `HUFL`, `HULL`, `MUFL`, `MULL`, `LUFL`, `LULL`: 負荷関連特徴量
- `OT`: オイル温度（予測対象）

主モデルの検証では、時間単位の予測 horizon を設定するため `ETTh1` / `ETTh2` を使用しました。
`ETTm1` / `ETTm2` はデータ構造・時間粒度の確認に使用しています。

## 分析フロー

1. データ品質確認
2. EDA
3. 過去OT・負荷・時間特徴量の検討
4. Naive Baselineの設定
5. Linear Regression / Random Forest / XGBoost / LightGBM の比較
6. 1 / 3 / 6 / 12 / 24時間先予測の比較
7. PoCとしての運用価値・限界の整理

## 予測設定

時刻 `t` までに利用可能な情報を用いて、将来の `OT` を予測します。

評価した予測時間：

- 1時間先
- 3時間先
- 6時間先
- 12時間先
- 24時間先

データ分割は時系列順に行っています。

- Train: 70%
- Validation: 15%
- Test: 15%

ランダム分割は使用していません。

## 比較したモデル

### Naive Baseline
`未来OT = 現在OT` とする単純予測です。
機械学習を使う価値があるか判断するための比較基準として使用しました。

### Linear Regression
単純で説明しやすく、過去OTとの強い関係をどこまで線形モデルで捉えられるか確認するため採用しました。

### Random Forest
非線形関係や特徴量間の相互作用を捉えることで、Linear Regressionを上回れるか確認しました。

### XGBoost
表形式データで高性能が期待できるBoosting系モデルとして比較しました。

### LightGBM
XGBoostと同様に、複雑な非線形モデルによる追加改善があるか確認しました。

## モデル選定

Random Forest / XGBoost / LightGBMも実装・比較しましたが、
ValidationでLinear Regressionを安定して上回る結果は得られませんでした。

そのため、

- 予測精度
- 説明性
- 再現性
- 実装・保守の簡潔さ

を総合し、Linear Regressionを主モデルとして採用しました。

ただし、Transformer 1の短期予測ではNaive Baselineが非常に強く、
機械学習が必ずしも有効ではない点も重要な結果として扱っています。

## 主な結果

### Transformer 1 (ETTh1)

| 予測先 | 選択構成 | Test MAE | Baseline MAE | 改善率 |
|---|---|---:|---:|---:|
| 1時間 | Naive Baseline | 0.4441 | 0.4441 | 0.0% |
| 3時間 | Naive Baseline | 0.8280 | 0.8280 | 0.0% |
| 6時間 | Naive Baseline | 1.2425 | 1.2425 | 0.0% |
| 12時間 | Linear Regression + OT/Lags | 1.5198 | 1.6598 | 8.4% |
| 24時間 | Linear Regression + All features | 1.8551 | 1.6999 | -9.1% |

Transformer 1では、短期ほど現在OTの継続性が強く、
機械学習による追加価値は限定的でした。

### Transformer 2 (ETTh2)

| 予測先 | 選択構成 | Test MAE | Baseline MAE | 改善率 |
|---|---|---:|---:|---:|
| 1時間 | Linear Regression + OT/Lags | 0.5671 | 0.9948 | 43.0% |
| 3時間 | Linear Regression + All features | 1.5933 | 2.7713 | 42.5% |
| 6時間 | Linear Regression + All features | 2.8110 | 4.8632 | 42.2% |
| 12時間 | Linear Regression + All features | 3.3984 | 6.8031 | 50.0% |
| 24時間 | Linear Regression + All features | 3.6689 | 3.6606 | -0.2% |

Transformer 2では1〜12時間先でBaselineを明確に上回りました。
一方、24時間先では優位性がほぼ消失しました。

## PoCとしての解釈

- 1〜3時間先：直近監視・早期警戒への活用候補
- 6〜12時間先：当日中の点検優先順位・運用判断への活用候補
- 24時間先：現状モデルでは明確な改善がなく、翌日単位の保全計画には追加改善が必要

また、設備によって予測特性が大きく異なるため、
一律のモデルではなく設備別評価が必要と考えられます。

## 限界

- ETT内の変数のみを使用している
- 外気温・天候・保全履歴などの外部要因を使用していない
- Transformer 1 / 2で予測性能が異なる
- 一部期間では高温サンプルが少ない
- 24時間先では明確な改善が見られない
- 実運用で必要な予測時間・許容誤差はクライアント要件に依存する

## 今後の改善

- 気象・外気温データの追加
- 設備別モデルの最適化
- 時系列Cross Validation
- 高温イベントの分類・早期警戒評価
- より長期の予測が必要な場合の専用時系列モデル検討

## リポジトリ構成

```text
ett-oil-temperature-poc/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── ETTh1.csv
│   ├── ETTh2.csv
│   ├── ETTm1.csv
│   └── ETTm2.csv
├── notebooks/
│   └── ETT_PoC_analysis.ipynb
└── report/
    └── （PoC報告スライドPDFを配置）
```

## 実行方法

### Google Colab
1. `notebooks/ETT_PoC_analysis.ipynb` をColabで開く
2. `data/` の4CSVを使用できる状態にする
3. 上から順番に全セルを実行する

### ローカル環境
```bash
pip install -r requirements.txt
jupyter notebook
```

その後、`notebooks/ETT_PoC_analysis.ipynb` を上から順に実行してください。

## データ出典

ETT Dataset  
https://github.com/zhouhaoyi/ETDataset
