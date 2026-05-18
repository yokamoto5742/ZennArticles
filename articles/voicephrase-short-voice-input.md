---
title: "Python + Google Cloud STT で自作した音声入力アプリを1日100回使ってわかったこと "
emoji: "🎙️"
type: "tech"
topics: ["python", "googlecloud", "windows", "音声入力"]
published: true
---

## はじめに: 普段使いして感じる音声入力のメリット

タイピングが得意な方は、音声入力よりも自分の手で打った方が速くて正確、と思われるかもしれません。

私は自作音声入力アプリ**VoicePhrase** を **1日100回以上** 使っていますが、実感としては入力速度は **「短文ではタイピングと同程度、文章が長くなるほど音声入力のほうが速くなる」** です。

音声入力のメリットは、入力速度よりも**文章を書くハードルが下がることで返信の長さや丁寧さがよくなる効果**が大きいと思われます。 

この記事では、**“文章を書くハードルを下げるツール”** としての音声入力を提案し、自作する際のポイントを共有します。

:::message
本記事で紹介する VoicePhrase は Windows 向けの デスクトップアプリケーション です。Google Cloud Speech-to-Text API を利用するため、利用には Google Cloud のアカウントが必要です。
:::

## なぜ自作したか — 他の音声入力アプリとの違い

既存の音声入力ツールをいくつか試しましたが、キーボードのタイピングに代わる普段使いできるツールをなかなか見つけられませんでした。

- **Windows 標準の音声入力は日本語認識精度が弱い** — 業務用語や固有名詞で誤認識して結局手で打ち直すことになる
- **ファイル名変更欄やダイアログに貼り付けられない** — 「テキスト入力できる場所」が限定される
- **クラウド型アプリはネット瞬断で音声が消えることがある** — Wi-Fi環境だと音声入力の最中にネットが切れて初めから言い直すケースがある

これらを解決するために、**自分が必要とする最小機能だけを組み合わせた専用ツール** を作ってみよう、という結論になりました。

設計のポイントは次の3点です。

1. **Google Cloud Speech-to-Text API** で日本語認識精度を確保
2. **貼り付け先を選ばない直接入力**
3. **ローカル WAV 保存** で通信瞬断に耐え、F8 キーで再送可能

## 使ってみる — クイックスタート

まずは動かせる状態まで持っていきます。事前に [uv](https://docs.astral.sh/uv/getting-started/installation/) のインストールが必要です。

### 1. クローンと依存パッケージ

```bash
git clone https://github.com/your-repo/VoicePhrase.git
cd VoicePhrase
uv sync
```

### 2. Google Cloud 認証情報を1行 JSON に変換

`.env` に `GOOGLE_CREDENTIALS_JSON` をセットしますが、サービスアカウントキーの JSON は改行を含むため、付属スクリプトで1行化します。

```bash
python scripts/json_minifier.py
```

ファイル選択ダイアログが開くので、Google Cloud Console からダウンロードした JSON を選択してください。タイムスタンプ付きの出力ファイル(例: `credentials_20240430_123456.json`)が生成されます。

### 3. `.env` を作成

```bash:.env
GOOGLE_PROJECT_ID=my-awesome-app-123456
GOOGLE_LOCATION=asia-northeast1
GOOGLE_CREDENTIALS_JSON={"type":"service_account","project_id":"...",...}
```

### 4. 起動

```bash
python main.py
```

Pause キーを押す → 話す → Pause キーで停止。これだけでアクティブウィンドウへテキストが貼り付けられます。

| キー | 機能 |
|------|------|
| Pause | 録音開始 / 停止 |
| F8 | 直前の音声を再変換(ネット瞬断時の再送) |
| F9 | 句読点機能の有効 / 無効 |
| Esc | アプリケーション終了 |

### 5.実行ファイルの生成

PyInstaller を使用して実行ファイルを生成します。

```bash
python build.py
```

## 設計のポイント

VoicePhrase の開発で特に時間のかかった設計のポイントです。

### ポイント1: 貼り付けの安定性の確保 — クリップボード検証とリトライ

最初はクリップボードへの書き込みを単純に行っていましたが、**クリップボードへの書き込みが実際に反映されていない**ケースがしばしば発生しました。

対策として、`service/paste_backend` では以下の2段階の処理を実装しています。

**① クリップボードへの書き込み検証（`safe_clipboard_copy`）**
書き込み直後に `pyperclip.paste()` で読み返し、内容が一致するかを確認します。
不一致の場合は最大3回までリトライし、それでも失敗した場合は `False` を返します。

**② キーストロークによる貼り付け（`safe_paste_text`）**
クリップボードへの書き込みを確認した後、`pynput` の `Controller` で`Ctrl+V` を送信します。

:::message
クリップボード経由の貼り付けは貼り付けるアプリ側の `Ctrl+V` 受け付け実装に依存するため、一部のウインドウでは動作しない場合があります。
:::

### ポイント2: ネット瞬断耐性 — ローカル WAV 保存 + F8 再送

クラウド型音声入力の大きなの弱点は「ネットワーク失敗時に音声が消える」ことです。例えば重要な会議の議事録を取っている最中にこれが起こると、大きな作業ロスが発生します。

VoicePhrase は録音した音声を **ローカルの WAV ファイルとして一時保存** してから API に送ります。送信が失敗した場合、ユーザーは **F8 キーを押すだけで直前の音声を再送** できます。

```
[マイク入力] → [WAV保存] → [STT API送信] → [テキスト貼り付け]
                  ↑              ↓ (失敗時)
                  └── F8で再送 ──┘
```

「発声し直さなくていい」という体験は、普段使いしていると予想以上に効いてきます。**このアプリなら文字起こしの失敗を気にせず話せる** という安心感につながりました。

### ポイント3: 【目玉】Tkinter + バックグラウンドスレッドの落とし穴 — UIQueueProcessor 導入

**Tkinter + 音声処理のアプリを自作する人には特に共有したい話題** です。

#### 初期実装で起きたこと

当初は録音処理・API 送信・結果表示をすべて別スレッドで動かしていました。

```python
# ❌ アンチパターン: 別スレッドから直接 Tkinter を更新
def on_transcription_done(text: str):
    self.label.config(text=text)  # 別スレッドから呼ばれる
    self.window.update()
```

これで **原因不明のフリーズが頻発** しました。再現条件が掴めず、長時間動かしていると突然アプリが落ちる。最初は Google Cloud API のタイムアウトを疑い、リトライ処理やタイムアウト値を変更しましたが、まったく改善しませんでした。

#### 原因: Tkinter はシングルスレッド前提

Tkinter のウィジェット操作は、メインスレッド（イベントループを回しているスレッド）からのみ呼ぶ前提で設計されています。別スレッドからの操作は、運が良ければ動き、運が悪ければデッドロックする、という不安定な挙動になります。

#### 解決: すべての UI 更新をキュー経由に統一

`UIQueueProcessor` という小さなクラスを導入し、**UI 更新は必ずキューに積み、メインスレッドの `after()` で逐次取り出して実行する** ように統一しました。

```python:app/ui_queue_processor.py
import queue
import threading
import tkinter as tk
from typing import Callable


class UIQueueProcessor:
    def __init__(self, root: tk.Tk):
        self.master = root
        self._ui_queue: queue.Queue = queue.Queue()
        self._ui_lock = threading.Lock()
        self._is_shutting_down = False

    def start(self) -> None:
        """ポーリングを開始する。インスタンス生成後に明示的に呼ぶこと。"""
        if self.is_ui_valid():
            self.master.after(50, self._process_queue)

    def schedule_callback(self, callback: Callable, *args) -> None:
        """別スレッドから安全に呼べる。コールバックと引数をキューに積む。"""
        if self._is_shutting_down:
            return
        self._ui_queue.put_nowait((callback, args))

    def _process_queue(self) -> None:
        processed = 0
        while processed < 10:  # 1回のポーリングで最大10件処理
            try:
                callback, args = self._ui_queue.get_nowait()
                callback(*args)  # 引数を展開して実行
                processed += 1
            except queue.Empty:
                break

        if not self._is_shutting_down and self.is_ui_valid():
            self.master.after(50, self._process_queue)

    def is_ui_valid(self) -> bool:
        if self._is_shutting_down:
            return False
        try:
            with self._ui_lock:
                return bool(self.master and self.master.winfo_exists())
        except Exception:
            return False

    def shutdown(self) -> None:
        self._is_shutting_down = True
```

呼び出し側はこうなります。

```python
# ✅ 別スレッドから UI を更新する正しい方法

# 初期化後に必ず start() を呼ぶ
ui_queue = UIQueueProcessor(root)
ui_queue.start()

# コールバックと引数を分けて渡せる
def on_transcription_done(text: str):
    ui_queue.schedule_callback(lambda: self.label.config(text=text))
```

この変更で **アプリのフリーズはほぼ消滅** しました。Tkinter + バックグラウンドスレッド構成のアプリを書くなら、最初からこのパターンを採用することをおすすめします。

:::message alert
「UI 更新を別スレッドから呼んでも動くことがある」のが落とし穴です。開発中は問題なくても本番運用で突然フリーズします。**全 UI 更新をキュー経由に統一する** というルールにしてください。
:::

## 認識精度を上げる小技 — 専門用語登録と置換ルール

専門用語は、汎用の音声認識モデルでは正しく拾えない場合があります。VoicePhrase は2段構えでこれを補正します。

### フレーズヒント(`data/technical_terms.txt`)

Google Cloud Speech-to-Text API の `speech_recognition_hints` に渡す専門用語リストです。1行1フレーズで登録します。

```text:data/technical_terms.txt
$OOV_CLASS_DIGIT_SEQUENCE
$OPERAND
加齢黄斑変性
```

`$OOV_CLASS_*` は Google STT API の **クラストークン** で、数字列や演算子といった「カテゴリ」を認識ヒントとして与えられます。業務で数字や記号の認識精度に困っている方はぜひ試してみてください。

### 後処理置換(`data/replacements.txt`)

誤認識するパターンが決まっている単語は、CSV 形式で後処理置換します。

```csv:data/replacements.txt
# 医療系の同音異義語を補正
焼死体,硝子体

# 不要な疑問符を句点に
?,。
```

フレーズヒントと置換ルールの二段構えにすることで、利用する環境に合わせて運用しながら少しずつ精度が向上していきます。

## 使い続けてわかったメリット

VoicePhrase を1日100回以上使うようになってタイピングの負荷が減り、文章を書く際に **「相手にどう伝えるか」により集中できる** ようになりました。その結果、メールなどにより適切な長さで丁寧な返信ができるようになりました。

また、生成 AI へのプロンプト入力も劇的に楽になりました。自分の意図を伝えるための長文プロンプトを書くハードルが下がり、AI の出力品質が上がる、という好循環が生まれています。

## おわりに

特に次のいずれかに当てはまる方は、ぜひ一度触ってみてください。

- 1日にチャットやメールの返信を大量に書いている方
- ファイル名やダイアログにも音声入力したい方
- ネット環境が不安定な場所で音声入力を使いたい方
- Tkinter + バックグラウンドスレッドのアプリを自作している方

VoicePhrase
https://github.com/yokamoto5742/VoicePhrase

想定外の使い方を見つけた方は、ぜひコメントで教えていただけると嬉しいです。
