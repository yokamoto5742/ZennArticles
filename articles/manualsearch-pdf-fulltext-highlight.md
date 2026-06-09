---
title: "ファイルを1つずつ開くのが大変だったので自作したPDF横断検索アプリ"
emoji: "🔍"
type: "tech"
topics: ["python", "windows", "全文検索"]
published: false
---

## はじめに

業務マニュアルの「あの記載、どこにあったっけ」を探すのに時間を取られていませんか。

私は Acrobat Reader で 1 ファイルずつ開いては `Ctrl + F`、を繰り返すことに限界を感じ、**PDF・テキスト・Markdown を横断して全文検索し、PDF は該当箇所まで Adobe Acrobat 上で自動ハイライト表示する** Windows デスクトップアプリ **ManualSearch** を自作しました。

この記事では、**既製ツールに限界を感じて自作しようとする方**に向けて、試行錯誤した**Adobe Acrobat の自動ハイライト連携**と**全文検索インデックス設計**を共有します。

:::message
ManualSearch は個人的に欲しくて作ったツールで、大規模な導入実績はまだありません。ソースコードは GitHub で公開しています。
:::

### 対象読者

- Acrobat Reader / Windows 検索 / Everything などを試したが、**PDF 本文の横断検索＋該当箇所ハイライト**で物足りなかった方
- 生成AIで業務マニュアルのRAGを試したが、回答精度がいまいちで現場で使えず困っている方
- Shift_JIS 混在の日本語ファイルで「検索できない」に繰り返しぶつかっている方

### 想定環境

- OS: Windows 10 / 11
- Python: 3.12 以上
- Adobe Acrobat Reader DC（PDF のハイライト表示に使用）

---

## 既製ツールとの比較

**複数フォルダにまたがる大量の PDF マニュアルから、本文を検索して該当箇所をすぐ開きたい**のに、ちょうど合うツールがなかなか見つかリませんでした

| ツール | 私の用途で足りない所                          |
| ---- |-------------------------------------|
| Acrobat Reader | 1 ファイルずつしか開けず、複数マニュアルの横断検索ができない     |
| Windows 検索 | 私の環境では PDF 本文の検索が遅く、ヒットしないこともあった    |
| Everything | ファイル名検索は非常に速いが、**本文検索とハイライトは守備範囲外** |

「**該当箇所を開いた瞬間にハイライトで分かる**」というツールが見当たらず、検索でファイルが見つかっても開いた後にもう一度 `Ctrl + F` をやり直す。この二度手間をなくしたいので、以下の要件を満たすものを自作することにしました。

- 複数フォルダ（共有フォルダ含む）を横断して全文検索する
- PDF は該当箇所まで自動ハイライトして開く
- 大量ファイルでも待たされないようインデックスで高速化する
- Shift_JIS 混在の日本語ファイルでも文字化けせず検索する

---

## ManualSearch の紹介

ManualSearch は PyQt 製の Windows デスクトップアプリです。

- 検索対象フォルダを複数登録（共有ドライブも指定可）
- 検索語を入力（カンマ区切りで複数語、AND / OR 切り替え）
- 結果をダブルクリックで開くと、PDF は Acrobat 上で該当箇所を開いてハイライトされる

技術的な構成はおおまかに次の通りです。

```
UI 層 (PyQt5 / Widgets)
  └─ 検索入力・結果表示・フォルダ管理・インデックス管理
コアロジック層 (service/)
  ├─ FileSearcher          … ThreadPoolExecutor による並列全文検索
  ├─ IndexedFileSearcher   … インデックスを使った高速検索
  ├─ SearchIndexer         … インデックス作成・差分更新
  └─ pdf_handler           … PDF テキスト抽出とハイライト
```

開発の工夫を2つ共有します。

---

## ポイント1: Adobe Acrobat の自動ハイライト連携

「検索語の箇所を開いた瞬間にハイライトしたい」。一見簡単そうですが、ここでなかなか時間がかかりました。

### 何が難しいのか

ManualSearch 自身は PDF ビューアを持っていません。

表示は外部の Adobe Acrobat に任せています。つまり「**別アプリに、特定の語をハイライトさせた状態で開かせる**」必要があります。Acrobat を外部から制御する話は公式ドキュメントが乏しく、ネット上の情報も断片的でした。

### 採ったアプローチ

最終的に落ち着いたのは、**ハイライトを焼き込んだ一時 PDF を生成し、それを Acrobat で開く**という方法です。Acrobat に検索を指示するのではなく、**こちらで先にハイライト済みの PDF を用意してしまう**わけです。

PDF のテキスト抽出とハイライト注入には [PyMuPDF (fitz)](https://pymupdf.readthedocs.io/) を使いました。検索語の出現位置を矩形（rect）として取得し、ハイライト注釈を追加します。

```python:service/pdf_handler.py（概念コード）
import fitz  # PyMuPDF

def highlight_pdf(pdf_path: str, search_terms: list[str]) -> str:
    doc = fitz.open(pdf_path)
    for page in doc:
        for term in search_terms:
            # ページ内で検索語の出現位置（矩形）をすべて取得
            for rect in page.search_for(term):
                page.add_highlight_annot(rect)
    # 元ファイルを汚さないよう、一時ファイルとして保存
    highlighted_path = make_temp_path(pdf_path)
    doc.save(highlighted_path)
    doc.close()
    return highlighted_path
```

生成した一時 PDF を、設定済みの Acrobat 実行ファイルパスで開きます。

```python:service/file_opener.py（概念コード）
import subprocess

def open_with_acrobat(acrobat_path: str, pdf_path: str) -> None:
    subprocess.Popen([acrobat_path, pdf_path])
```

### 詰まった点と回避策

公式情報が乏しいぶん、実際に動かして初めて分かった落とし穴がありました。

- **元ファイルを直接書き換えない**
  ハイライトを元の PDF に追記してしまうと、共有フォルダ上の資料を破壊しかねません。必ず一時ファイルに保存し、元ファイルは読み取り専用で扱います。

- **一時ファイルが溜まり続ける**
  開くたびに一時 PDF が生成されるため、クリーンアップの仕組みが要ります。

- **Acrobat の実行パスが環境で違う**
  `C:\Program Files\Adobe\Acrobat DC\Acrobat\Acrobat.exe` は環境やバージョンで変わります。ハードコードせず、設定ファイル（`config.ini`）で差し替えられるようにしました。

```ini:config.ini
[Paths]
acrobat_path = C:\Program Files\Adobe\Acrobat DC\Acrobat\Acrobat.exe
```

:::message
「アプリ内に PDF ビューアを実装する」より「使い慣れた Acrobat に開かせる」ほうが、業務現場では受け入れられやすいと判断しました。ビューアの自作はメンテコストが高く、ユーザーも普段使いのソフトで開けるほうが安心だからです。
:::

---

## ポイント2: インデックス設計と差分更新

もう一つの山が、検索速度です。

### なぜインデックスが必要か

愚直に実装すると、検索のたびに対象フォルダの全ファイルを開き、テキストを抽出し、マッチング——を毎回行うことになります。ファイルが数十なら許容できますが、数百の PDF が相手だと待ち時間が一気に膨らみます。

そこで、**事前にファイルの中身を抽出してインデックス化**し、検索時はインデックスだけを見るようにしました。

```python:service/search_indexer.py（概念コード）
indexer = SearchIndexer("search_index.json")
indexer.create_index(
    directories=["C:/Documents"],
    include_subdirs=True,
)
```

### 設計判断: 何を選び、何を捨てたか

インデックスは凝りだすときりがありません。専用の検索エンジン（全文検索ライブラリや外部 DB）を導入する選択肢もありました。しかし今回は、**個人〜小規模チームが追加インストールなしで使えること**を優先し、あえて素朴な構成にしています。

- **永続化形式は JSON**
  追加の DB を要求せず、中身を人間が確認・デバッグしやすい。大規模検索には不利ですが、想定する規模では十分でした。

- **差分更新で再インデックスを避ける**
  毎回ゼロから作り直すと、ファイルが増えるほど更新が重くなります。**ファイルの修正時刻とハッシュ値**を記録しておき、変わったファイルだけ再処理します。

```python:service/search_indexer.py（概念コード）
# 修正時刻 → 変わっていそうなファイルを絞り込む
# ハッシュ値 → 本当に中身が変わったかを確定する
indexer.update_index(
    directories=["C:/Documents"],
    include_subdirs=True,
)
```

修正時刻だけでは「時刻は変わったが中身は同じ」を取りこぼせないため、最終確認にハッシュ値を使う二段構えにしています。

- **インデックス破損時はフォールバック**
  JSON が壊れたり古かったりした場合に検索ごと失敗すると、ユーザーは何が起きたか分かりません。インデックスが使えないときは、**その場でファイルを直接検索する方式に自動で切り替える**ようにしました。速度は落ちても、結果は返る——という安全側に倒しています。

:::message
「速いが壊れると何も返らない」より「遅くても必ず返る」を選びました。業務ツールでは、ピーク性能よりも壊れ方の優しさのほうが効いてくると考えています。
:::

---

## 日本語の壁: Shift_JIS 混在の文字化け

最後に、短いけれど日本語環境では避けて通れない話を一つ。

現場の古いテキストファイルは Shift_JIS と UTF-8 が混在しがちで、エンコーディングを決め打ちすると簡単に文字化けします。文字化けしたテキストは当然、検索にもヒットしません。

ManualSearch では [chardet](https://pypi.org/project/chardet/) でエンコーディングを自動判定してから読み込むことで、この「検索できない」を防いでいます。

```python:service/content_extractor.py（概念コード）
import chardet

def read_text(path: str) -> str:
    raw = open(path, "rb").read()
    encoding = chardet.detect(raw)["encoding"] or "utf-8"
    return raw.decode(encoding, errors="replace")
```

地味ですが、日本語のマニュアルを横断検索するうえでは効果が大きい一手でした。

---

## 使ってみたい人へ

ManualSearch は GitHub で公開しています。

https://github.com/

:::message alert
冒頭でも触れた通り、これは私が自分用に作った段階のツールです。動作には Windows・Python 3.12 以上・Adobe Acrobat Reader DC が必要です。実運用での検証はこれからの部分も多いので、使ってみて気づいた点や改善案があればフィードバックいただけると嬉しいです。
:::

導入手順はリポジトリの README にまとめていますが、おおまかには次の通りです。

```bash
git clone <repository-url>
cd ManualSearch
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
python main.py
```

---

## おわりに

「資料が探せない」という日常の小さなストレスは、既製ツールを探し回るだけでなく、**自分の手で解決する**という選択肢もあります。

そして自作してみると、表向きは単純な「検索してハイライトする」機能の裏に、

- 外部 Acrobat をどう制御するか
- インデックスをどう壊れにくく差分更新するか
- 日本語のエンコーディングをどう吸収するか

といった、調べても答えの出にくいハマりどころが潜んでいました。同じように既製ツールに物足りなさを感じている方の、実装の手がかりになれば幸いです。
