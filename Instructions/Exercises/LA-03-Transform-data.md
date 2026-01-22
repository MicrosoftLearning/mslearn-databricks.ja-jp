---
lab:
  title: Azure Databricks で Apache Spark を使用してデータを変換する
---

# Azure Databricks で Apache Spark を使用してデータを変換する

Azure Databricks は、一般的なオープンソース Databricks プラットフォームの Microsoft Azure ベースのバージョンです。 Azure Databricks は Apache Spark 上に構築されており、ファイル内のデータの操作を伴うデータ エンジニアリングおよび分析タスクに対して非常にスケーラブルなソリューションを提供します。

Azure Databricks の一般的なデータ変換タスクには、データのクリーニング、集計の実行、型キャストなどがあります。 これらの変換は、分析のためのデータ準備に不可欠であり、大規模な ETL (抽出、変換、読み込み) プロセスの一部です。

この演習の所要時間は約 **30** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

1. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

1. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

1. リポジトリをクローンした後、次のコマンドを入力して **setup-serverless.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```
    ./mslearn-databricks/setup-serverless.ps1
    ```

1. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
1. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの記事「[Azure Databricks での探索的データ分析](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/)」を確認してください。

## Azure Databricks ワークスペースを開く

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

## ノートブックを作成する

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。

1. 既定のノートブック名 (**無題のノートブック *[日付]***) を `Transform data with Spark` に変更し、**[接続]** ドロップダウン リストで、**[サーバーレス]** コンピューティングがまだ選択されていない場合は選択します。 コンピューティングが実行されていない場合は、開始に 1 分ほどかかる場合があります。

## データの取り込み

1. ノートブックの最初のセルに、次のコードを入力して、数個のラボ ファイルを格納するためのボリュームを作成します。

    ```sql
    %sql
    CREATE VOLUME IF NOT EXISTS spark_lab
    ```

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。このコードでは、*Python* を使用して GitHub からボリュームにデータ ファイルをダウンロードします。

    ```python
    import requests

    # Define the current catalog
    catalog_name = spark.sql("SELECT current_catalog()").collect()[0][0]

    # Define the base path using the current catalog
    volume_base = f"/Volumes/{catalog_name}/default/spark_lab"

    # List of files to download
    files = ["2019.csv", "2020.csv", "2021.csv"]

    # Download each file
    for file in files:
        url = f"https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/{file}"
        response = requests.get(url)
        response.raise_for_status()

        # Write to Unity Catalog volume
        with open(f"{volume_base}/{file}", "wb") as f:
            f.write(response.content)
    ```

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 出力の下で、**[+ コード]** アイコンを使用して新しいセルを追加し、それを使用してデータのスキーマを定義する次のコードを実行します。

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   orderSchema = StructType([
        StructField("SalesOrderNumber", StringType()),
        StructField("SalesOrderLineNumber", IntegerType()),
        StructField("OrderDate", DateType()),
        StructField("CustomerName", StringType()),
        StructField("Email", StringType()),
        StructField("Item", StringType()),
        StructField("Quantity", IntegerType()),
        StructField("UnitPrice", FloatType()),
        StructField("Tax", FloatType())
   ])
   df = spark.read.load(f'/Volumes/{catalog_name}/default/spark_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

## データをクリーンアップする

このデータセットの **Tax** 列に重複する行と `null` 値があることを確認します。 そのため、データに対してさらなる処理と分析を行う前に、クリーニングの手順が必要です。

1. 新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、テーブルから重複する行を削除し、`null` エントリを正しい値に置き換えます。

    ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
    display(df.limit(100))
    ```

**[税]** 列の値を更新した後、そのデータ型が再び `float` に設定されていることを確認します。 これは、計算の実行後にデータ型が `double` に変更されるためです。 `double` は `float` よりもメモリ使用量が多いため、列を型キャストして `float` に戻す方がパフォーマンスに優れています。

## データフレームをフィルター処理する

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。これにより以下が実行されます。
    - 販売注文データフレームの列をフィルター処理して、顧客名とメール アドレスのみを含める。
    - 注文レコードの合計数をカウントする
    - 個別の顧客の数をカウントする
    - 個別の顧客を表示する

    ```python
   customers = df['CustomerName', 'Email']
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    次の詳細を確認します。

    - データフレームに対して操作を実行すると、その結果として新しいデータフレームが作成されます (この場合、df データフレームから列の特定のサブセットを選択することで、新しい customers データフレームが作成されます)
    - データフレームには、そこに含まれているデータの集計やフィルター処理に使用できる count や distinct などの関数が用意されています。
    - `dataframe['Field1', 'Field2', ...]` 構文は、列のサブセットを簡単に定義できる方法です。 また、**select** メソッドを使用すると、上記のコードの最初の行を `customers = df.select("CustomerName", "Email")` のように記述することができます

1. 次は新しいコード セルで次のコードを実行して、特定の製品の注文を行った顧客のみを含めるフィルターを適用してみましょう。

    ```python
   customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    複数の関数を "チェーン化" すると、1 つの関数の出力が次の関数の入力になることに注意してください。この場合、select メソッドによって作成されたデータフレームは、フィルター条件を適用するために使用される where メソッドのソース データフレームとなります。

## データフレーム内のデータを集計してグループ化する

1. 新しいコード セルで次のコードを実行して、注文データを集計およびグループ化します。

    ```python
   productSales = df.select("Item", "Quantity").groupBy("Item").sum()
   display(productSales)
    ```

    その結果が、製品ごとにグループ化された注文数の合計を示していることに注意してください。 **groupBy** メソッドを使用すると、*Item* ごとに行がグループ化されます。その後の **sum** 集計関数は、残りのすべての数値列に適用されます (この場合は *Quantity*)

1. 新しいコード セルで、他の集計を試してみましょう。

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

    今回の結果には 1 年あたりの販売注文数が表示されます。 select メソッドには *"OrderDate"* フィールドの year コンポーネントを抽出するための SQL **year** 関数が含まれており、抽出された year 値に列名を割り当てるために **alias** メソッドが使用されていることに注意してください。 次に、データが派生 *Year* 列によってグループ化され、各グループの行**数**が計算されます。その後、結果として生成されたデータフレームを並べ替えるために、最後に **orderBy** メソッドが使用されます。

> **注**:Azure Databricks での DataFrame の使用について詳しくは、Azure Databricks ドキュメントの [DataFrame の概要 - Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) に関するページを参照してください。

## SQL コードをセル内で実行する

1. PySpark コードが含まれているセルに SQL ステートメントを埋め込むことができるのは便利ですが、データ アナリストにとっては、SQL で直接作業できればよいという場合も多くあります。 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```python
   df.createOrReplaceTempView("salesorders")
    ```

このコード行は、SQL ステートメントで直接使用できる一時ビューを作成します。

1. 新しいセルで次のコードを実行します。
   
    ```python
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    次の点に注意してください。
    
    - セルの先頭にある **%sql**行 (magic と呼ばれます) は、このセル内でこのコードを実行するには、PySpark ではなく、Spark SQL 言語ランタイムを使用する必要があることを示しています。
    - SQL コードにより、前に作成した **salesorder** ビューが参照されます。
    - SQL クエリからの出力は、セルの下に自動的に結果として表示されます。
    
> **注**: Spark SQL とデータフレームの詳細については、[Spark SQL のドキュメント](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html)を参照してください。

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
