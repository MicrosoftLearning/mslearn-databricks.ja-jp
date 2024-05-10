---
lab:
  title: Azure Databricks で Delta Lake を使用する
---

# Azure Databricks で Delta Lake を使用する

Delta Lake は、データ レイクの上に Spark 用のトランザクション データ ストレージ レイヤーを構築するためのオープンソース プロジェクトです。 Delta Lake では、バッチ データ操作とストリーミング データ操作の両方にリレーショナル セマンティクスのサポートが追加され、Apache Spark を使用して、データ レイク内の基になるファイルに基づくテーブル内のデータを処理しクエリを実行できる *Lakehouse* アーキテクチャを作成できます。

このラボは完了するまで、約 **40** 分かかります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、Azure portal で、Azure Databricks ワークスペースを対話形式で作成してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示されたら、***PowerShell*** 環境を選んで、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

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

1. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。
1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime のバージョン**: 13.3 LTS (Spark 3.4.1、Scala 2.12) 以降
    - **Photon Acceleration を使用する**: 選択済み
    - **ノードの種類**: Standard_DS3_v2
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成してデータを取り込む

次に、Spark ノートブックを作成し、この演習で使用するデータをインポートしましょう。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を "**差分レイクの探索**" に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。
1. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

    ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/products.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv
    ```

1. そのセルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行します。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、ファイルからデータを読み込み、最初の 10 行を表示します。

    ```python
   df = spark.read.load('/delta_lab/products.csv', format='csv', header=True)
   display(df.limit(10))
    ```

## ファイル データをデルタ テーブルに読み込む

データはデータフレームに読み込まれています。 これを差分テーブルに保持してみましょう。

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```python
   delta_table_path = "/delta/products-delta"
   df.write.format("delta").save(delta_table_path)
    ```

    差分レイク テーブルのデータは Parquet 形式で保存されます。 ログ ファイルも作成され、データに加えられた変更が追跡されます。

1. 新しいコード セルを追加し、それを使用して次のシェル コマンドを実行して、差分データが保存されているフォルダーの内容を表示します。

    ```
    %sh
    ls /dbfs/delta/products-delta
    ```

1. Delta 形式のファイル データは、**DeltaTable** オブジェクトに読み込むことができます。これを使用して、テーブル内のデータを表示および更新できます。 新しいセルで次のコードを実行してデータを更新し、製品 771 の価格を 10% 下げます。

    ```python
   from delta.tables import *
   from pyspark.sql.functions import *
   
   # Create a deltaTable object
   deltaTable = DeltaTable.forPath(spark, delta_table_path)
   # Update the table (reduce price of product 771 by 10%)
   deltaTable.update(
       condition = "ProductID == 771",
       set = { "ListPrice": "ListPrice * 0.9" })
   # View the updated data as a dataframe
   deltaTable.toDF().show(10)
    ```

    更新は差分フォルダー内のデータに保持され、その場所から読み込まれたすべての新しいデータフレームに反映されます。

1. 次のコードを実行して、差分テーブル データから新しいデータフレームを作成します。

    ```python
   new_df = spark.read.format("delta").load(delta_table_path)
   new_df.show(10)
    ```

## ログと *time-travel* を確認する

データの変更がログされるため、Delta Lake の *time-travel* 機能を使用して、以前のバージョンのデータを表示できます。 

1. 新しいコード セルで次のコードを使用して、製品データの元のバージョンを表示します。

    ```python
   new_df = spark.read.format("delta").option("versionAsOf", 0).load(delta_table_path)
   new_df.show(10)
    ```

1. ログには、データに対する変更の完全な履歴が含まれています。 次のコードを使用して、過去 10 件の変更のレコードを確認します。

    ```python
   deltaTable.history(10).show(10, False, True)
    ```

## カタログ テーブルを作成する

ここまで、テーブルの基になった Parquet ファイルが含まれるフォルダーからデータを読み込むことで、デルタ テーブルを操作しました。 データをカプセル化する "カタログ テーブル" を定義し、SQL コードで参照できる名前付きテーブル エンティティを提供できます。** Spark では、デルタ レイク用に次の 2 種類のカタログ テーブルがサポートされています。

- テーブル データを含む Parquet ファイルへのパスで定義される "外部" テーブル。**
- Spark プール用の Hive メタストアで定義されている "*マネージド*" テーブル。

### 外部テーブルを作成する

1. このコードを使用して、**AdventureWorks** という名前の新しいデータベースを作成し、先ほど定義した Dalta ファイルへのパスに基づいて、そのデータベース内に **ProductsExternal** という外部テーブルを作成します。

    ```python
   spark.sql("CREATE DATABASE AdventureWorks")
   spark.sql("CREATE TABLE AdventureWorks.ProductsExternal USING DELTA LOCATION '{0}'".format(delta_table_path))
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsExternal").show(truncate=False)
    ```

    新しいテーブルの **Location** プロパティが、指定したパスであることに注意してください。

1. 次のコードを使用してテーブルにクエリを実行します。

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsExternal;
    ```

### マネージド テーブルを作成する

1. 次のコードを実行して、(製品 771 の価格を更新する前に) 最初に **products.csv** ファイルから読み込んだデータフレームに基づいて、**ProductsManaged** という名前のマネージド テーブルを作成 (および記述) します。

    ```python
   df.write.format("delta").saveAsTable("AdventureWorks.ProductsManaged")
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsManaged").show(truncate=False)
    ```

    このテーブルで使用される Parquet ファイルのパスは指定しませんでした。これは Hive メタストアで管理され、テーブルの説明の **Location** プロパティ (**dbfs:/user/hive/warehouse/** パス内) に表示されます。

1. 次のコードを使用して、マネージド テーブルにクエリを実行します。構文はマネージド テーブルの場合と同じであることに注意してください。

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsManaged;
    ```

### 外部テーブルとマネージド テーブルを比較する

1. 次のコードを使用して、**AdventureWorks** データベースのテーブルを一覧表示します。

    ```sql
   %sql
   USE AdventureWorks;
   SHOW TABLES;
    ```

1. ここで次のコードを使用して、これらのテーブルが基づくフォルダーを確認します。

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

1. 次のコードを使用して、データベースから両方のテーブルを削除します。

    ```sql
   %sql
   USE AdventureWorks;
   DROP TABLE IF EXISTS ProductsExternal;
   DROP TABLE IF EXISTS ProductsManaged;
   SHOW TABLES;
    ```

1. ここで次のコードを含むセルを再実行して、差分フォルダーの内容を表示します。

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

    マネージド テーブルのファイルは、テーブルが削除されると自動的に削除されます。 ただし、外部テーブルのファイルは残ります。 外部テーブルを削除すると、データベースからテーブル メタデータのみが削除されます。データ ファイルは削除されません。

1. 次のコードを使用して、**products-delta** フォルダー内の差分ファイルに基づく新しいテーブルをデータベースに作成します。

    ```sql
   %sql
   USE AdventureWorks;
   CREATE TABLE Products
   USING DELTA
   LOCATION '/delta/products-delta';
    ```

1. 次のコードを使用して新しいテーブルにクエリを実行します。

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM Products;
    ```

    テーブルは既存の差分ファイル (ログされた変更履歴を含む) に基づいているため、これには製品データに以前行った変更が反映されています。

## ストリーミング データにデルタ テーブルを使用する

Delta Lake では、"*ストリーミング*" データがサポートされています。 デルタ テーブルは、Spark 構造化ストリーミング API を使用して作成されたデータ ストリームの "シンク" または "ソース" に指定できます。** ** この例では、モノのインターネット (IoT) のシミュレーション シナリオで、一部のストリーミング データのシンクにデルタ テーブルを使用します。 シミュレートされたデバイス データは、次のような JSON 形式です。

```json
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"ok"}
{"device":"Dev2","status":"error"}
{"device":"Dev1","status":"ok"}
{"device":"Dev1","status":"error"}
{"device":"Dev2","status":"ok"}
{"device":"Dev2","status":"error"}
{"device":"Dev1","status":"ok"}
```

1. 新しいセルで、次のコードを実行して JSON ファイルをダウンロードします。

    ```bash
    %sh
    rm -r /dbfs/device_stream
    mkdir /dbfs/device_stream
    wget -O /dbfs/device_stream/devices1.json https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/devices1.json
    ```

1. 新しいセルで、次のコードを実行して、JSON デバイス データを含むフォルダーに基づいてストリームを作成します。

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   # Create a stream that reads data from the folder, using a JSON schema
   inputPath = '/device_stream/'
   jsonSchema = StructType([
   StructField("device", StringType(), False),
   StructField("status", StringType(), False)
   ])
   iotstream = spark.readStream.schema(jsonSchema).option("maxFilesPerTrigger", 1).json(inputPath)
   print("Source stream created...")
    ```

1. 新しいコード セルを追加し、それを使用して、データのストリームをデルタ フォルダーに永続的に書き込みます。

    ```python
   # Write the stream to a delta table
   delta_stream_table_path = '/delta/iotdevicedata'
   checkpointpath = '/delta/checkpoint'
   deltastream = iotstream.writeStream.format("delta").option("checkpointLocation", checkpointpath).start(delta_stream_table_path)
   print("Streaming to delta sink...")
    ```

1. 他の差分フォルダーと同様に、データを読み取るコードを追加します。

    ```python
   # Read the data in delta format into a dataframe
   df = spark.read.format("delta").load(delta_stream_table_path)
   display(df)
    ```

1. 次のコードを追加して、ストリーミング データの書き込み先となる差分フォルダーに基づいてテーブルを作成します。

    ```python
   # create a catalog table based on the streaming sink
   spark.sql("CREATE TABLE IotDeviceData USING DELTA LOCATION '{0}'".format(delta_stream_table_path))
    ```

1. 次のコードを使用してテーブルにクエリを実行します。

    ```sql
   %sql
   SELECT * FROM IotDeviceData;
    ```

1. 次のコードを実行して、新しいデバイス データをストリームに追加します。

    ```Bash
    %sh
    wget -O /dbfs/device_stream/devices2.json https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/devices2.json
    ```

1. 次の SQL クエリ コードを再実行して、新しいデータがストリームに追加され、差分フォルダーに書き込まれたことを確認します。

    ```sql
   %sql
   SELECT * FROM IotDeviceData;
    ```

1. 次のコードを実行してストリームを停止します。

    ```python
   deltastream.stop()
    ```

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。