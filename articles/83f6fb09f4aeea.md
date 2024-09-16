---
title: "Cloud Run Job を使って並列バッチ処理を楽に実装する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudrun", "gcp", "batch", "python"]
published: true
published_at: 2024-09-17 08:00 # 予約投稿（投稿日時を指定する）
publication_name: gixo # Publication投稿をする際のみ指定する
---

## はじめに

バッチ処理の実装において、依存関係のない処理を高速化するために並列化することは一般的です。
しかし、Cloud Run Service を使って、並列処理を実装するとなると、実装が複雑になりがちです。そこで、使ったことのなかった Cloud Run Job を使って並列バッチ処理を簡単に実装できるか検証してみました。

## TL;DR

Cloud Run には、HTTP リクエストを受け付ける Cloud RUn サービスと、定期的なジョブを実行する Cloud Run ジョブ の 2 つのモードがあります。このリポジトリでは、バッチ処理のモードである Cloud Run Jobs を使って BigQuery のデータを定期的にエクスポートする処理を検証しました。

単純なバッチ処理なら、並列処理のバッチ処理を組むことができました。ただし、並列数と対象テーブル数が一致していない場合、思ったように処理が完了しないため、実際にバッチ処理を組む場合は工夫が必要そうです。

## Cloud Run Jobs とは

とりあえず Cloud Run Jobs って何

|                | Service                                                       | Jobs                                                           |
| -------------- | ------------------------------------------------------------- | -------------------------------------------------------------- |
| 実行方法       | HTTP リクエストをトリガーとしてコンテナを実行する             | コンソールや gcloud コマンド、Google Cloud APIs 経由で実行する |
| 実行タイミング | HTTP リクエストをトリガーとして実行する                       | 任意のタイミングで実行できる                                   |
| 機能           | ウェブ リクエスト、イベント、関数に応答するコードの実行に使用 | 非同期タスクを実行するための機能を備えている                   |

詳しくは、こちらを参考
https://zenn.dev/google_cloud_jp/articles/cloudrun-jobs-basic

## 検証の流れ

### 出力先の Cloud Storage バケットを作成する

```bash
gcloud storage buckets create gs://test-cloud-run-jobs \
  --location=us-central1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access
```

### サービスアカウントを作成

```bash
gcloud iam service-accounts create cloud-run-jobs-sa \
  --display-name "Cloud Run Jobs Service Account"
```

### サービスアカウントに必要な権限を付与

必要な権限は以下の通りです。

```bash
gcloud projects add-iam-policy-binding {project_id} \
  --member=serviceAccount:cloud-run-jobs-sa@{project_id}.iam.gserviceaccount.com \
  --role=roles/bigquery.admin
gcloud projects add-iam-policy-binding {project_id}  \
  --member=serviceAccount:cloud-run-jobs-sa@{project_id}.iam.gserviceaccount.com \
  --role=roles/storage.objectAdmin
gcloud projects add-iam-policy-binding {project_id} \
  --member=serviceAccount:cloud-run-jobs-sa@{project_id}.iam.gserviceaccount.com \
  --role=roles/logging.logWriter
gcloud projects add-iam-policy-binding {project_id} \
  --member=serviceAccount:cloud-run-jobs-sa@{project_id}.iam.gserviceaccount.com \
  --role=roles/storage.admin
```

### 実装

BigQuery のデータをエクスポートする処理を実装します。

大まかな処理の流れは以下の通りです。

- BigQuery のデータセットに含まれるテーブル名のリストを取得
- GCS にエクスポートされた前回分のファイルを削除
- BigQuery のテーブルを GCS にエクスポート

並列処理にするために工夫した部分は、以下の通りです。
タスクは 30 並列で実行しますが、それぞれが別のテーブルを処理するように、task_index を使用しています。
例： `task_index = int(os.environ.get("CLOUD_RUN_TASK_INDEX", 0))`
そして、`tables[task_index]` で リストのうち task_index 番目のテーブルを処理しています。

難点は、並列数とテーブル数が一致していない場合、思ったように処理が完了しないことです。
例えば、テーブル数が 10 で並列数が 30 の場合、20 個のタスクがエラーになります。
逆に、テーブル数が 30 で並列数が 10 の場合、20 個のタスクが何もせずに終了します。

テーブル数が変更になっても、並列数を変更しなくてもいいように、実際にバッチ処理を組む場合は工夫が必要そうです。

```python: main.py
import os  # noqa: D100

from google.cloud import bigquery, logging, storage  # noqa: D100

# ロギングクライアントを作成
logging_client = logging.Client()
logger = logging_client.logger("export-table-to-gcs")


def get_tables(bq_client: bigquery.Client, dataset_id: str) -> list[str]:
    """Get table IDs in the dataset.

    Args:
        bq_client (bigquery.client): BigQuery client
        dataset_id (str): Dataset ID

    Returns:
        list[str]: Table IDs
    """
    try:
        dataset_ref = bq_client.dataset(dataset_id)
        tables = bq_client.list_tables(dataset_ref)
    except Exception as e:
        logger.log_text(f"Failed to list tables in {dataset_id}: {e}", severity="ERROR")
        raise e

    return sorted([table.table_id for table in tables])


def export_table_to_gcs(
    bq_client: bigquery.Client,
    dataset_id: str,
    table_id: str,
    destination_uri: str,
) -> None:
    """Export BigQuery table to GCS.

    Args:
        bq_client (bigquery.client): BigQuery client
        dataset_id (str): Dataset ID
        table_id (str): Table ID
        destination_uri (str): GCS URI
    """
    try:
        dataset_ref = bq_client.dataset(dataset_id)
        table_ref = dataset_ref.table(table_id)
    except Exception as e:
        logger.log_text(f"Failed to get table reference: {e}", severity="ERROR")
        raise e

    try:
        extract_job = bq_client.extract_table(
            table_ref,
            destination_uri,
            location="US",
        )
        extract_job.result()  # 処理が完了するまで待機
    except Exception as e:
        logger.log_text(f"Failed to extract {table_id}: {e}", severity="ERROR")
        raise e
    logger.log_text(f"Exported {table_id} to {destination_uri}", severity="INFO")


def cleanup_gcs_files(gcs_client: storage.Client, bucket_name: str, table_name: str) -> None:
    """Delete all files in GCS.

    Args:
        gcs_client (storage.Client): GCS client
        bucket_name (str): Bucket name
        table_name (str): Table name
    """
    try:
        bucket = gcs_client.get_bucket(bucket_name)
        blobs = bucket.list_blobs(prefix=f"{table_name}/")
        if not blobs:
            logger.log_text(f"No files in {bucket_name}", severity="INFO")
            return
        for blob in blobs:
            blob.delete()
    except Exception as e:
        logger.log_text(f"Failed to delete files in {bucket_name}: {e}", severity="ERROR")
        raise e
    logger.log_text(f"Deleted all files in {bucket_name}", severity="INFO")


def main() -> str:
    """Export a table to GCS.

    Returns:
        str: A message
    """
    project_id = os.environ.get("PROJECT_ID")
    dataset_id = os.environ.get("SOURCE_DATASET")
    bucket_name = os.environ.get("DESTINATION_BUCKET")

    bq_client = bigquery.Client()
    gcs_client = storage.Client()

    logger.log_text(f"Connected to BigQuery: {bq_client}", severity="INFO")
    print(f"project_id: {project_id}")
    print(f"source_dataset: {dataset_id}")
    print(f"bq_client: {bq_client}")
    tables = get_tables(bq_client, dataset_id)
    logger.log_text(f"Tables in {dataset_id}: {tables}", severity="INFO")

    # TASK_INDEX環境変数でタスクのインデックスを取得
    task_index = int(os.environ.get("CLOUD_RUN_TASK_INDEX", 0))
    logger.log_text(f"Task index: {task_index}", severity="INFO")

    # タスクインデックスに対応するテーブルを取得
    table_id = tables[task_index]
    # 前回のエクスポートファイルを削除
    cleanup_gcs_files(gcs_client, bucket_name, table_id)
    # エクスポートファイルは1ファイル1GBまでなので、複数ファイルに分割してエクスポート
    destination_uri = f"gs://{bucket_name}/{table_id}/{table_id}_*.csv"
    export_table_to_gcs(bq_client, dataset_id, table_id, destination_uri)
    logger.log_text(f"Exported {table_id} to {destination_uri}", severity="INFO")

    return f"Exported {table_id} to {destination_uri}"


if __name__ == "__main__":
    main()

```

その他のファイルは、Github にアップロードしています。
[test-cloud-run-jobs](https://github.com/y-tsuritani/test-cloud-run-jobs)

### Cloud Run にデプロイ

```bash
gcloud builds submit ./src --tag gcr.io/{project_id}/test-cloud-run-job

gcloud run jobs create export-bq-to-gcs-job \
  --image=gcr.io/{project_id}/test-cloud-run-job \
  --memory=512Mi \
  --region=us-central1 \
  --service-account=cloud-run-jobs-sa \
  --set-env-vars=PROJECT_ID={project_id},SOURCE_DATASET=TEST_DATA,DESTINATION_BUCKET=test-cloud-run-jobs \
  --tasks 30
```

2 回目以降は、`gcloud run jobs create` ではなく `gcloud run jobs update`で更新できます。

### ダミーデータを用意する

ダミーデータを用意します。中身は何でもいいです。
適当に csv ファイルを 30 個作成して、GCS にアップロードします。
その後、csv ファイルを BigQuery にロードします。

csv 用のバケットを作成しておきます。

```bash
gsutil mb gs://test-source-csv
```

```bash
cd data
bash create_dummy_tables.sh
```

### Cloud Run Jobs を実行

Cloud Run Jobs を実行します。

```bash
gcloud run jobs execute export-bq-to-gcs-job --region us-central1
```

実行に成功すると以下のようなメッセージが表示されます。

```bash
app-py3.12tyj app % gcloud run jobs execute export-bq-to-gcs-job --region us-central1
✓ Creating execution... Done.
  ✓ Provisioning resources...
Done.
Execution [export-bq-to-gcs-job-mwxpd] has successfully started running.

View details about this execution by running:
gcloud run jobs executions describe export-bq-to-gcs-job-mwxpd

Or visit https://console.cloud.google.com/run/jobs/executions/details/us-central1/export-bq-to-gcs-job-mwxpd/tasks?
```

出力されたリンクをクリックすると、ジョブの実行状況を確認できます。
Cloud Run Service で並列処理を実行するとステータスを確認するのが面倒なので、Cloud Run Jobs を使うと GUI で簡単に確認ができて便利そうです。

![実行状況](https://storage.googleapis.com/zenn-user-upload/756be7b66df8-20240916.png)

### エクスポートされたファイルを確認

エクスポートされたファイルを確認してみましたが、無事にエクスポートされていました。
実行履歴を確認しましたが、ちゃんと並列で処理されてそうでした。

### 資材を削除

検証が終わったら、忘れずに資材を削除します。

### Next Step

今回の検証には含めませんでしたが、もう少し複雑なバッチ処理を組む場合は、Workflows から Cloud Run Jobs を呼び出すことが可能です。
`googleapis.run.v1.namespaces.jobs.run` で Cloud Run Jobs を呼び出すことができます。
実行時には環境変数の上書きができるため、テーブル名やエクスポート先のバケット名を動的に変更することも可能です。

[公式ドキュメント](https://cloud.google.com/workflows/docs/tutorials/execute-cloud-run-jobs?hl=ja#deploy-workflow)

### おわりに

Cloud Run Jobs を使って BigQuery のデータを定期的にエクスポートするバッチ処理を検証しました。
単純なバッチ処理なら、並列処理のバッチ処理を組むことができました。ただし、並列数と対象テーブル数が一致していない場合、思ったように処理が完了しないため、実際にバッチ処理を組む場合は工夫が必要そうなことがわかりました。
並列処理は、バッチ処理を設計する上で重要な要素なので、今後も検証を続けていきたいと思います。
