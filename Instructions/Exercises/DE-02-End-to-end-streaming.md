---
lab:
  title: Azure Databricks の Delta Live Tables を使用したエンド ツー エンドストリーミング パイプライン
---

# Azure Databricks の Delta Live Tables を使用したエンド ツー エンドストリーミング パイプライン

Azure Databricks で Delta Live Tables を使用してエンドツーエンドのストリーミング パイプラインを作成するには、データの変換を定義する必要があります。この変換は、Delta Live Tables がタスク オーケストレーション、クラスター管理、監視を通じて管理します。 このフレームワークでは、継続的に更新されるデータを処理するためのストリーミング テーブル、複雑な変換用の具体化されたビュー、中間変換とデータ品質チェック用のビューがサポートされています。

このラボは完了するまで、約 **30** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている可能性があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal で ***PowerShell*** 環境を選択し、新しい Cloud Shell を作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;**、**&#10530;**、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりできます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトが完了するまで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの[Delta Lake の概要](https://docs.microsoft.com/azure/databricks/delta/delta-intro)に関する記事をご確認ください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS 以降のランタイム バージョンを持つクラスターが既にある場合は、それを使ってこの演習を完了し、この手順をスキップできます。

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します (**[その他]** サブメニューを確認する必要がある場合があります)。

1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime のバージョン**: 13.3 LTS (Spark 3.4.1、Scala 2.12) 以降
    - **Photon Acceleration を使用する**: 選択済み
    - **ノード タイプ**: Standard_D4ds_v5
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

    > **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成してデータを取り込む

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。
2. 既定のノートブック名 (**無題のノートブック *[日付]***) を **Delta Live Tables Ingestion** に変更します。

3. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/device_stream
    mkdir /dbfs/device_stream
    wget -O /dbfs/device_stream/device_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/device_data.csv
     ```

4. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## ストリーミング データにデルタ テーブルを使用する

Delta Lake では、"*ストリーミング*" データがサポートされています。 デルタ テーブルは、Spark 構造化ストリーミング API を使用して作成されたデータ ストリームの "シンク" または "ソース" に指定できます。** ** この例では、モノのインターネット (IoT) のシミュレーション シナリオで、一部のストリーミング データのシンクにデルタ テーブルを使用します。 次のタスクでは、この差分テーブルはリアルタイムでデータ変換のソースとして機能します。

1. 新しいセルで、次のコードを実行して、CSV デバイス データを含むフォルダーに基づいてストリームを作成します。

     ```python
    from pyspark.sql.functions import *
    from pyspark.sql.types import *

    # Define the schema for the incoming data
    schema = StructType([
        StructField("device_id", StringType(), True),
        StructField("timestamp", TimestampType(), True),
        StructField("temperature", DoubleType(), True),
        StructField("humidity", DoubleType(), True)
    ])

    # Read streaming data from folder
    inputPath = '/device_stream/'
    iotstream = spark.readStream.schema(schema).option("header", "true").csv(inputPath)
    print("Source stream created...")

    # Write the data to a Delta table
    query = (iotstream
             .writeStream
             .format("delta")
             .option("checkpointLocation", "/tmp/checkpoints/iot_data")
             .start("/tmp/delta/iot_data"))
     ```

2. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。

この差分テーブルは、リアルタイムでデータ変換のソースになります。

   > 注: 上記のコード セルはソース ストリームを作成します。 したがって、ジョブの実行は完了状態に変更されることはありません。 ストリーミングを手動で停止するには、新しいセルで `query.stop()` を実行します。
   
## Delta Live Tables パイプラインを作成する

パイプラインは、Delta Live Tables でデータ処理ワークフローの構成と実行に使用されるメイン ユニットです。 Python または SQL で宣言された有向非巡回グラフ (DAG) を使用して、データ ソースをターゲット データセットにリンクします。

1. 左側のサイドバーで **Delta Live Tables** を選択し、 **[パイプラインの作成]** を選択します。

2. **[パイプラインの作成**] ページで、次の設定を使用して新しいパイプラインを作成します。
    - **パイプライン名**: `Ingestion Pipeline`
    - **製品エディション**: 詳細
    - **パイプライン モード**: トリガー
    - **ソース コード**: *空白のままにします*
    - **ストレージ オプション**: Hive メタストア
    - **保存場所**: `dbfs:/pipelines/device_stream`
    - **ターゲット スキーマ**: `default`

3. **[作成]** を選択してパイプラインを作成します (パイプライン コード用の空のノートブックも作成されます)。

4. パイプラインが作成されたら、右側のパネルの**ソース コード**の下にある空白のノートブックへのリンクを開きます。 これにより、新しいブラウザー タブでノートブックが開きます。

    ![delta-live-table-pipeline](./images/delta-live-table-pipeline.png)

5. 空白のノートブックの最初のセルで、次のコードを入力して (ただし実行はしません) Delta Live Tables を作成し、データを変換します。

     ```python
    import dlt
    from pyspark.sql.functions import col, current_timestamp
     
    @dlt.table(
        name="raw_iot_data",
        comment="Raw IoT device data"
    )
    def raw_iot_data():
        return spark.readStream.format("delta").load("/tmp/delta/iot_data")

    @dlt.table(
        name="transformed_iot_data",
        comment="Transformed IoT device data with derived metrics"
    )
    def transformed_iot_data():
        return (
            dlt.read("raw_iot_data")
            .withColumn("temperature_fahrenheit", col("temperature") * 9/5 + 32)
            .withColumn("humidity_percentage", col("humidity") * 100)
            .withColumn("event_time", current_timestamp())
        )
     ```

6. ノートブックを含むブラウザー タブを閉じて (内容が自動的に保存されます)、パイプラインに戻ります。 **[開始]** を選択します。

7. パイプラインが正常に完了したら、最初に作成した最新の **Delta Live Tables のインジェスト**に戻り、新しいセルで次のコードを実行して、指定した保存場所に新しいテーブルが作成されたことを確認します。

     ```sql
    %sql
    SHOW TABLES
     ```

## 結果を視覚化として表示する

テーブルを作成した後、テーブルをデータフレームに読み込み、データを視覚化することができます。

1. 最初のノートブックで、新しいコード セルを追加し、次のコードを実行して `transformed_iot_data` をデータフレームに読み込みます。

    ```python
    %sql
    SELECT * FROM transformed_iot_data
    ```

1. 結果の表の上にある **[+]** 、 **[視覚化]** の順に選択して視覚化エディターを表示し、次のオプションを適用します。
    - **視覚化の種類**: 折れ線
    - **X 列**: タイムスタンプ
    - **Y 列**: *新しい列を追加して、**temperature_fahrenheit** を選択します*。 ****合計***集計を適用します*。

1. 視覚化を保存し、結果のグラフをノートブックに表示します。
1. 新しいコード セルを追加し、次のコードを入力してストリーミング クエリを停止します。

    ```python
    query.stop()
    ```
    

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
