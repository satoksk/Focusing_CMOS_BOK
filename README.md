# Focusing_CMOS_BOK

CMOS カメラを用いた BOK 望遠鏡のフォーカス評価システムです。  
画像から星を抽出し、FWHM・楕円率・位置角などを測定し、  
アクチュエータの位置と FWHM の関係を可視化します。

---

## 🚀 機能

- FITS 画像の読み込みと group_id ごとの整理
- PSF スムージング＋背景除去
- Photutils による星検出と FWHM 測定
- Sigma-clipping による外れ値除去
- Dask による並列処理（大量画像でも高速）
- アクチュエータ位置（a,b,c）との関係プロット
- スプライン補間による “ベストフォーカス” 推定

---

## 📁 ディレクトリ構造（例）
Focusing_CMOS_BOK/
├─ main.py # エントリポイント
├─ focusing.py # meta 入力と CSV 読み書き
├─ data/
│ ├─ FOCUSING_20250915/ # FITS ファイル
│ └─ focus_meta.csv # アクチュエータ位置
├─ README.md
└─ requirements.txt
