# svd
参考文献: Imada et al 2015 "Monthly Weather Review"

目的: 上記論文中のmethodologyの再現

データ: CMIP6データセット(42モデル)内のSSTとPRCP

データ期間：1851-2014年

SST: 4月の月平均(各モデルにおいて同グリッド内の1851-2014年データを用いて正規化している)。解像度5x5°

PRCP: 5月- 10月までの月平均値、SST同様正規化を行っている。解像度1x1°

## 手法の概要

1. SST(インド洋・太平洋, `sst[..., 6:18, 9:57]`, 12x48格子)を陸域マスクして行列 X(空間, 時間) に変換
2. タイ周辺の降水(20x20格子)を行列 Y(空間, 時間) に変換
3. 共分散行列 `C_XY = X @ Y.T / N` を特異値分解し、左右特異行列 P, Q を得る
4. 予測時は SST を P に射影した時間展開係数 `A = P.T @ sst` を求め、`sum_i Q[:, i] * A[i]` を指定モード数だけ足し合わせて降水を再構成する

## ディレクトリ構成

```
svd/
├── main.ipynb            # 最初の試行。1モデル(番号1)の直近30年でSVDし、再現実験と別モデルへの外挿を確認
├── svd_modes.ipynb       # 40モデル x 165年でSVD。内挿実験、モード別特異ベクトルの図示、外挿実験
├── forecast_test.ipynb   # 学習/検証データ(pickle)でSVDし、1000検証サンプルの予測を保存する(評価の入力を作る)
└── evaluation/           # forecast_test.ipynb の保存結果を読み込んで評価する
    ├── intergration.ipynb            # 評価の統合版。MAE, RMSE地図, 相関地図, GMMによるTrue/False判定, ROC/AUC
    ├── roc_curve_plot.ipynb          # ROC曲線(サンプル単位のパーセンタイル)
    ├── diff_evaluation.ipynb         # 格子ごとの平均絶対誤差の棒グラフ
    └── correaltion_coefficient.ipynb # 格子ごとの相関係数の平均
```

## 入出力パス (実行マシン上の絶対パス)

すべて `/docker/mnt/d/research/D2/cnn3` 以下を参照します。

| 種類 | パス | 使用ノートブック |
| --- | --- | --- |
| 入力 SST | `predictors/predictors_coarse_std_Apr_o.npy` | main, svd_modes |
| 入力 PRCP | `predictant/continuous/pr_1x1_std_MJJASO_thailand.npy` | main, svd_modes |
| 入力 学習/検証セット | `test/continuous/seed1_predictors_coarse_std_Apr_msot-pr_1x1_std_MJJASO_thailand.pickle` | forecast_test, evaluation/* |
| 出力 予測 | `result/svd/thailand/1x1/seed1_mode{mode}.npy` | forecast_test が保存、evaluation/* が読込 |
| 出力 正規化予測 | `result/svd/thailand/1x1/seed1_mode{mode}_norm.pickle` | forecast_test が保存、evaluation/* が読込 |

`{mode}` は予測に使うモード数です (forecast_test では `mode_fin`、評価側では `mode` 変数)。

## 実行順序

1. `forecast_test.ipynb` を上から実行し、最後のセルで `seed1_mode{mode_fin}.npy` と `_norm.pickle` を保存
2. `evaluation/` 内のノートブックで `mode` を保存時の `mode_fin` と合わせて実行 (主に `intergration.ipynb`)

`main.ipynb` と `svd_modes.ipynb` は単独で実行できる探索用ノートブックです。

## 必要なパッケージ

numpy, matplotlib, cartopy, scipy, scikit-learn, colormaps (`intergration.ipynb` の地図描画で使用)

## 注意点 (既知の問題)

コードセルは出力との対応を保つため変更していません。以下は実データで確認してください。

* `forecast_test.ipynb` の「1000validationの保存」セルで、外側ループと内側ループが同じ変数 `i` を使っています。そのため `validation_output[i, :]` は常に最後の行 (`mode_fin - 1`) にしか書き込まれず、他の行は `np.empty` の未初期化値のままです。保存済みの `seed1_mode*.npy` と、それを使う評価結果は見直しが必要な可能性があります。内側ループの変数名を例えば `m` に変えれば意図通りになります。
* `svd_modes.ipynb` の内挿実験で SST 側の reshape が `40*165` と固定値になっています。`train_int`, `train_fin`, `training_period` を変えると PRCP 側 (`(train_fin-train_int)*abs(training_period)`) と食い違います。
* `forecast_test.ipynb` と `svd_modes.ipynb` の `mask` は陸域を 0 で埋めるため、`latlon_to_X` の NaN 判定では陸域格子が除外されず、0 のまま行列に含まれます (`main.ipynb` は NaN で埋めるので除外されます)。
* `evaluation/intergration.ipynb` の `bimodal_dist` では2つ目の正規分布にも `gmm.weights_[0]` を掛けています (`weights_[1]` の意図かもしれません)。
* `evaluation/intergration.ipynb` の `stats.norm.interval(alpha=...)` は新しい SciPy では `confidence=...` に変わっています。`plt.cm.get_cmap` も新しい Matplotlib では削除されています。
* `evaluation/intergration.ipynb` の `correlation` は相関係数を小数第2位、`correaltion_coefficient.ipynb` は第3位で丸めてから平均しているため、両者の値は少し異なります。
