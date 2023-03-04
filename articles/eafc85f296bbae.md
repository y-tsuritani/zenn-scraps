---
title: "ChatGPT API を使って Cloud Functions で ChatGPT風SlackBot を作ってみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "slack", "cloudfunctions", "chatgpt", "openai"]
published: true
---
## 💡はじめに

2023年3月2日に **OpenAI** より ChatGPT に使われている言語モデル **`gpt-3.5-turbo`** を有料で利用できる **Chat Completion API** が公開されました🎊
早速、**Slack で ChatGPT との会話ができる Slack Bot** を作ってみました。

[使い方] Bot にメンションを飛ばすと ChatGPT風に返答してくれます。
![animation](https://storage.googleapis.com/zenn-user-upload/e3c9fcdf5fc4-20230304.gif)

## 📐アーキテクチャ

FaaS を利用すると、簡単なアプリケーションを気軽に構築できます。
今回は、GCP を使いたかったので **Cloud Functions[第2世代]** を利用しています。
**SlackBot** にメンション付きで質問すると、Cloud Functions が起動し、**Chat Completion API** を利用してGPT-3.5 に質問を送信し、受信した回答を Slack にメンション付きで返信します。

![architecture_img](https://storage.googleapis.com/zenn-user-upload/a5535da0a2d5-20230304.png)

## 🛠構築

Slack app を作成したことがない人でも作成できることを目指しているため、
Slack app の作成から Cloud Functions のデプロイまで細かく説明しています。

SlackBot のコードだけ確認したい人は、[コード](###コード) まで飛ばしてください。

### Slack app の作成

[Slack api](https://api.slack.com/) にアクセスして Slack app を作成してください。

![slack_create_app](https://storage.googleapis.com/zenn-user-upload/3a4f20c79b44-20230304.png)

from scratch を選択

![slack_create_from_scratch](https://storage.googleapis.com/zenn-user-upload/c384fafd50e6-20230304.png)

Basic Information で Bots, Event Subscriptions, Permissions を設定

![slack_create_bot](https://storage.googleapis.com/zenn-user-upload/b42fa9fa67e9-20230304.png)

OAuth & Permissions で Bot User OAuth Token を発行する

![slack_OAuth_token](https://storage.googleapis.com/zenn-user-upload/fccb23f34091-20230304.png)

Scopes でアプリが持つ権限を設定する
**Bot Token Scopes**

最低限

- app_mentions:read
- chat:write

今後は会話継続機能を実装したいのでこちらも

- channels:history
- chat:write.customize
- groups:history

### OpenAI API キーの取得

[OpenAI の HP](https://platform.openai.com/) にアクセスして OpenAI API key を取得してください。

![openai_api_key](https://storage.googleapis.com/zenn-user-upload/f9f8ea66098b-20230304.png)

### Cloud Functions のデプロイ

[Google Cloud Platform](https://console.cloud.google.com/?hl=JA) にアクセスして チュートリアルを参考にして Cloud Functions 関数を作成してください。

[公式ドキュメント：Cloud Functions のチュートリアル](https://cloud.google.com/functions/docs/tutorials/http?hl=ja)

以下のコードを Cloud Functions[第2世代] にデプロイしてください。
ランタイム設定は `Python3.10` です。
なお、発行した API キーは **Secret Manager** に登録して環境変数として参照します。

[公式ドキュメント：Cloud Functions でのシークレットの使用](https://cloud.google.com/functions/docs/configuring/secrets?hl=ja)

#### コード

https://github.com/y-tsuritani/SlackChatbot_like_ChatGPT

```python:main.py
import json
import logging
import os
import re
from typing import Union

import functions_framework
import google.cloud.logging
import openai
from box import Box
from flask import Request
from slack_bolt import App, context
from slack_bolt.adapter.google_cloud_functions import SlackRequestHandler

# Google Cloud Logging クライアント ライブラリを設定
logging_client = google.cloud.logging.Client()
logging_client.setup_logging(log_level=logging.DEBUG)

# 環境変数からシークレットを取得
slack_token = os.environ.get("SLACK_BOT_TOKEN")
openai_api_key = os.environ.get("OPENAI_API_KEY")
openai.api_key = openai_api_key

# FaaS で実行する場合、応答速度が遅いため process_before_response は True でなければならない
app = App(token=slack_token, process_before_response=True)
handler = SlackRequestHandler(app)


# Bot アプリにメンションしたイベントに対する応答
@app.event("app_mention")
def handle_app_mention_events(body: dict, say: context.say.say.Say):
    """アプリへのメンションに対する応答を生成する関数

    Args:
        body: HTTP リクエストのボディ
        say: 返信内容を Slack に送信
    """
    logging.debug(type(body))
    logging.debug(body)
    box = Box(body)
    user = box.event.user
    text = box.event.text
    only_text = re.sub("<@[a-zA-Z0-9]{11}>", "", text)
    # TODO: 単発の質問に返信するのみで、会話の履歴を参照する機能は未実装
    message = [{"role": "user", "content": only_text}]
    logging.debug(only_text)

    # OpenAI から AIモデルの回答を生成する
    (openai_response, total_tokens) = create_completion(message)
    logging.debug(openai_response)
    logging.debug(f"total_tokens: {total_tokens}")

    # 課金額がわかりやすいよう消費されたトークンを返信に加える
    say(f"<@{user}> {openai_response}\n消費されたトークン:{total_tokens}")


def create_chat_completion(messages: list) -> tuple[str, int]:
    """OpenAI API を呼び出して、質問に対する回答を生成する関数

    Args:
        messages: チャット内容のリスト

    Returns:
        GPT-3.5 の生成した回答内容
    """
    # openai の GPT-3 モデルを使って、応答を生成する
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",  # gpt-3.5-turbo を指定すると ChatGPT に近い文章が生成されます
        messages=messages,
        # TODO: max_tokens を指定すると token over になる
        # max_tokens=4096,  # 生成する応答の長さ 大きいと詳細な回答が得られますが、多くのトークンを消費します
        stop=None,
    )
    openai_response = response["choices"][0]["message"]["content"]
    total_tokens = response["usage"]["total_tokens"]
    # 応答のテキスト部分を取り出して返す
    return (openai_response, total_tokens)

# Cloud Functions で呼び出されるエントリポイント
@functions_framework.http
def slack_bot(request: Request):
    """slack のイベントリクエストを受信して各処理を実行する関数

    Args:
        request: Slack のイベントリクエスト

    Returns:
        SlackRequestHandler への接続
    """
    header = request.headers
    logging.debug(f"header: {header}")
    body = request.get_json()
    logging.debug(f"body: {body}")

    # URL確認を通すとき
    if body.get("type") == "url_verification":
        logging.info("url verification started")
        headers = {"Content-Type": "application/json"}
        res = json.dumps({"challenge": body["challenge"]})
        logging.debug(f"res: {res}")
        return (res, 200, headers)
    # 応答が遅いと Slack からリトライを何度も受信してしまうため、リトライ時は処理しない
    elif header.get("x-slack-retry-num"):
        logging.info("slack retry received")
        return {"statusCode": 200, "body": json.dumps({"message": "No need to resend"})}

    # handler への接続 class: flask.wrappers.Response
    return handler.handle(request)
```

```txt:requirements.txt
functions-framework==3.3.0
openai == 0.27.0 # ChatCompletion は 0.27.0 以降で対応
slack-bolt == 1.16.1
google-cloud-logging == 3.5.0
flask == 2.2.2
python-box == 7.0.0
```

### 呼び出し URL の設定
Cloud Functions の呼び出し URL を Slack app に設定する
（URL を入力するとテストが送信され、確認済みになると「Verified」に✅がつきます）
![slack_event_url](https://storage.googleapis.com/zenn-user-upload/4933cf2b5b72-20230304.png)

Subscribe to bot events で **app_mention** を設定

![slack_event_appmention](https://storage.googleapis.com/zenn-user-upload/3769fb868d22-20230305.png)

これで、アプリにメンションを送信すると、Cloud Functions が起動するようになります。

最後に、ワークスペースにアプリをインストールしましょう。

アプリの設定が完了したので、SlackBot にメンションを送ってみましょう。

![slack_install_app](https://storage.googleapis.com/zenn-user-upload/9b24d9005d08-20230305.png)

問題なければ、しばらくして返信が返ってくるはずです。

![chat_test](https://storage.googleapis.com/zenn-user-upload/2869bc9a0693-20230305.png)

## 📌おわりに

API キーの発行やら SlackApp の準備やらは少し面倒ですが、Cloud Functions を利用すればかなり簡単に **ChatGPT風の SlackBot** が作成できました。
実際の ChatGPT のようにセッションの会話履歴から継続して回答を生成してくれる機能は実装できていないので、次の段階は**セッション機能の実装**ですね。
ChatGPT API はかなり安価なのでぜひ皆さんも試してみてください。

読んでくださってありがとうございました。
https://twitter.com/tsunotto


## 📕API ドキュメント

### Slack

- [Slack Bolt 入門ガイド](https://slack.dev/bolt-python/ja-jp/tutorial/getting-started)
- [slack_bolt documentation](https://slack.dev/bolt-python/api-docs/slack_bolt/)
- [Standard app mention when your app is already in channel](https://api.slack.com/events/app_mention#app_mention-event__example-event-payloads__app-mention-that-invites-your-app-to-a-channel)

### GCP

- [Cloud Functions のチュートリアル](https://cloud.google.com/functions/docs/tutorials/http?hl=ja)
- [Cloud Functions でのシークレットの使用](https://cloud.google.com/functions/docs/configuring/secrets?hl=ja)

### OpenAI API

- [OpenAI API Reference](https://platform.openai.com/docs/api-reference/introduction)
- [Authentication](https://platform.openai.com/docs/api-reference/authentication)
- [Chat completions](https://platform.openai.com/docs/guides/chat/chat-completions-beta)
- [Completions](https://platform.openai.com/docs/api-reference/completions)
