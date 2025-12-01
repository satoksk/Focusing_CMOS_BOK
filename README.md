# Focusing_CMOS_BOK

CMOS カメラを用いた BOK 望遠鏡のフォーカス評価プロジェクト。  
FITS 画像から星を検出し、FWHM・楕円率などを測定、  
さらにアクチュエータ位置 (a, b, c) との関係から最適フォーカス位置を推定します。

---

## 📁 Repository Structure


Focusing_CMOS_BOK/
├─ FOCUSING_20250915/ # 観測データ（FITS）
├─ Test/ # テスト用スクリプト・資料
├─ Focusing.ipynb # メイン解析ノートブック
├─ focus_meta.csv # アクチュエータ位置 (a,b,c) の入力データ
├─ .gitignore
└─ README.md # このファイル


---

## 🚀 Overview

このプロジェクトでは以下を行います：

- FITS 画像の読み込み（group_id ごと）
- 背景除去 + PSF スムージング
- Photutils による星検出 (`SourceCatalog`)
- FWHM, ellipticity, PA の計算
- Dask を用いた高速並列処理
- Sigma-clipped 平均 FWHM の算出
- アクチュエータ平均距離 (ave) との関係プロット
- スプライン補間による Best-focus 推定

これらの処理はすべて **Focusing.ipynb** にまとまっています。

---

## 🔧 Dependencies

以下の Python ライブラリを使用します：

- numpy  
- pandas  
- astropy  
- photutils  
- scipy  
- matplotlib  
- dask[distributed]  
- tqdm  

## 📘 How to Use

### 1. データの配置

`FOCUSING_20250915/` ディレクトリに FITS ファイルを配置します。  
ファイル名規則：
...<group_id><index>.fits
..._<group_id>_meta.json

例：focustest_gcode4_5000_20250915_00012_4.fits
focustest_gcode4_5000_20250915_00012_meta.json


`group_id`（例：`00012`）ごとにまとめられ、  
`index` は CMOS 番号（0〜N）に対応します。

---

### 2. Jupyter Notebook を実行

メインノートブック：Focusing.ipynb

内容の流れ：

1. **FITS 読み込み**  
   `images_dict[group_id] → (n_frames, ny, nx)`

2. **星検出（Photutils）**  
   - 背景推定 (`Background2D`)  
   - PSF スムージング (`IntegratedGaussianPRF`)  
   - `detect_sources` によるセグメンテーション  
   - `SourceCatalog` により
     - x,y 座標  
     - FWHM  
     - a/b ratio  
     - Position Angle  
     を取得

3. **Dask による高速並列処理**  
   全 group_id × CMOS 画像を一括計算。
   ## ⚡ Parallel Processing with Dask

  本プロジェクトでは、複数の CMOS フレームに対して同じ detection を繰り返し実行するため、  
  `dask.distributed` を使ってローカルマシン上で並列実行しています。
  
  ```python
  from dask.distributed import Client
  import time
  
  client = Client(processes=True, n_workers=14, threads_per_worker=2)
  print(client)
  
  nrows, ncols = 4, 5
  
  t0 = time.time()
  
  det_results = run_detection_on_all_dask(
      images_dict,
      nrows=nrows,
      ncols=ncols,
      sigma=5.0,
      threshold_sigma=5.0,
      pixel_scale=0.23,
  )
  
  t1 = time.time()
  print(f"\n>>> detection finished in {t1 - t0:.3f} seconds")
  ```
🧠 何をしているコードか

Client(processes=True, n_workers=14, threads_per_worker=2)

ローカルマシン上に 14 プロセス × 各 2 スレッド の Dask ワーカーを立ち上げる

合計「最大 28 並列タスク」まで同時実行可能

run_detection_on_all_dask(...)

images_dict に含まれる全ての (group_id, frame) に対して
detection() を並列で実行し、結果を集約する関数

t0 / t1

全 detection にかかった 実行時間（秒） を測定してログに出力

🛠 環境に応じた Dask 設定の目安

Client(processes=True, n_workers=..., threads_per_worker=...) のパラメータは
CPU コア数とメモリ量 に応じて調整するのがポイントです。

1. n_workers（プロセス数）

目安：物理コア数と同じか、少し少なめ

例：

8 コア CPU → n_workers=6〜8

16 コア CPU → n_workers=12〜16

I/O 待ちが多い処理なら多めでもよいですが、
今回のような CPU バウンドな画像処理 では「物理コア ≒ worker 数」が無難です。

2. threads_per_worker（1ワーカーあたりのスレッド数）

目安：1〜2

画像処理は GIL の影響や NumPy の内部スレッドとの兼ね合いもあるため、
多すぎるスレッドは逆にオーバーヘッドになることがあります。

よく使う設定パターン：

threads_per_worker=1 → worker 数を増やして並列度を上げる

threads_per_worker=2 → 多少の並列 I/O / NumPy 内部スレッドを許容

3. メモリの考え方

Dask の Client 自体にはメモリを直接指定していませんが、

1 画像のメモリサイズ × 同時に処理するタスク数

＋ Background2D や中間配列（PSF・segmentation map など）

によってピークメモリが決まります。

例えば：

1 フレームが 10000×2560 の float64 → 約 200 MB / frame

同時に 10〜20 フレームを扱うと、それだけで数 GB 消費

もしメモリ不足を感じたら：

n_workers を減らす（同時実行数を落としてピークメモリを抑える）

もしくは、画像を float32 に落とすなども検討可能です。

🔁 典型的な設定例

デスクトップ (8 コア / 32 GB RAM)
```
client = Client(processes=True, n_workers=6, threads_per_worker=1)
```
ワークステーション (16 コア / 128 GB RAM)
```
client = Client(processes=True, n_workers=14, threads_per_worker=2)
```
ラップトップ (4 コア / 16 GB RAM)
```
client = Client(processes=True, n_workers=3, threads_per_worker=1)
```

5. **FWHM の sigma-clipped 平均の計算**  
   (`astropy.stats.SigmaClip`)

6. **アクチュエータ位置 (a,b,c) の入力 / CSV 読み込み**  
   - `focus_meta.csv` があれば読み込み  
   - なければ手動入力  
   - 自動的に `ave = (a+b+c)/3` を計算

7. **FWHM vs ave のプロット**  
   - 各 CMOS の散布図 + エラーバー  
   - 2次スプライン補完曲線  
   - Best-focus 位置の自動推定

## 🔭 Detection Pipeline Overview

このプロジェクトでは、Photutils を用いた  
**Gaussian smoothing ＋ Segmentation-based 星検出** により、  
各 CMOS フレームの星から **FWHM**・**楕円率**・**位置角** を測定します。

本パイプラインは *PSF フィッティング* を行わず、  
**2 次モーメントに基づく高速・安定な形状推定** を特徴とし、  
大量の CMOS データを高速処理するために最適化されています。

---

## ⚙️ Detection Method

1. **背景推定（Background2D）**  
   - `MedianBackground` によるローカル背景推定  
   - 大スケールの勾配やノイズを除去

2. **PSF smoothing（Gaussian kernel）**  
   - `CircularGaussianSigmaPRF(sigma)` により Gaussian kernel を生成  
   - 星像を強調するために平滑化

3. **星検出（Segmentation method）**  
   - `detect_sources()` により信号領域を抽出  
   - 閾値 = `threshold_sigma × background_rms`  
   - `npixels=5` で小さすぎる検出を除外

4. **星形状推定（SourceCatalog）**  
   - 2 次モーメントから以下を計算  
     - 半長軸 σ：`semimajor_sigma`  
     - 半短軸 σ：`semiminor_sigma`  
     - 軸比：`b/a`  
     - 位置角：`orientation`  
   - FWHM（pix → arcsec）は以下で計算：
     ```
     FWHM = 2 * sqrt(2 ln 2) * sqrt(a_sigma * b_sigma)
     ```

5. **CMOS Grid 分割（任意）**  
   - CMOS を `nrows × ncols` に分割  
   - 星を grid ID に分類して局所的な FWHM 解析も可能

6. **結果出力（pandas DataFrame）**  
   各検出天体について：

   | Column | 内容 |
   |--------|------|
   | xcentroid | 星の中心 x 座標 |
   | ycentroid | 星の中心 y 座標 |
   | segment_flux | セグメント内の総フラックス |
   | a_sigma / b_sigma | 半長軸・半短軸 |
   | fwhm_pix | ピクセル単位 FWHM |
   | fwhm_arcsec | 角秒単位 FWHM |
   | b/a | 軸比 |
   | pa_deg | 位置角（度） |

---

## 📄 Detection Code

```python
from photutils.psf import CircularGaussianSigmaPRF
from astropy.convolution import convolve

def detection(data, nrows, ncols,
              sigma=5.0,
              threshold_sigma=5.0,
              pixel_scale=0.23):

    # 背景推定
    bkg = Background2D(
        data,
        box_size=50,
        filter_size=3,
        bkg_estimator=MedianBackground()
    )
    data_sub = data - bkg.background

    # PSF smoothing kernel
    psf_model = CircularGaussianSigmaPRF(sigma=sigma)
    y, x = np.mgrid[-5:6, -5:6]
    psf = psf_model(x, y)
    psf /= np.sum(psf)

    smoothed = convolve(data_sub, psf)

    # 星検出
    threshold = threshold_sigma * bkg.background_rms_median
    segm = detect_sources(smoothed, threshold, npixels=5)

    # 星の shape 計算
    cat = SourceCatalog(data_sub, segm)

    a_sigma = cat.semimajor_sigma.value
    b_sigma = cat.semiminor_sigma.value
    sigma_mean = np.sqrt(a_sigma * b_sigma)

    fwhm_pix = sigma_mean * (2 * np.sqrt(2 * np.log(2)))
    fwhm_arcsec = fwhm_pix * pixel_scale

    pa_deg = np.degrees(cat.orientation.to("rad").value)

    det_smooth = pd.DataFrame({
        "xcentroid": cat.xcentroid,
        "ycentroid": cat.ycentroid,
        "segment_flux": cat.segment_flux,
        "a_sigma": a_sigma,
        "b_sigma": b_sigma,
        "fwhm_pix": fwhm_pix,
        "fwhm_arcsec": fwhm_arcsec,
        "b/a": b_sigma / a_sigma,
        "pa_deg": pa_deg,
    })

    return segm, cat, det_smooth



