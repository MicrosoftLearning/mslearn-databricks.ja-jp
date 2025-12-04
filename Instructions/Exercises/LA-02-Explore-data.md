---
lab:
  title: Azure Databricks を使用したデータ探索
---

# Azure Databricks を使用したデータ探索

Azure Databricks は、一般的なオープンソース Databricks プラットフォームの Microsoft Azure ベースのバージョンです。 

Azure Databricks を使用すると、探索的データ分析 (EDA) が容易になり、ユーザーは分析情報をすばやく検出し、意思決定を促すことができます。 統計手法や視覚化など、EDA のさまざまなツールや手法と統合して、データ特性を要約し、基になる問題を特定します。

この演習の所要時間は約 **30** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 

十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

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

1. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```
    ./mslearn-databricks/setup.ps1
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
   
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を `Explore data with Spark` に変更し、**[接続]** ドロップダウン リストで、**[サーバーレス SQL ウェアハウス]** がまだ選択されていない場合は選択します。 コンピューティングが実行されていない場合は、開始に 1 分ほどかかる場合があります。

## データの取り込み

1. ノートブックの最初のセルに、次のコードを入力して、数個のラボ ファイルを格納するためのボリュームを作成します。

    ```sql
    %sql
    CREATE VOLUME IF NOT EXISTS spark_lab
    ```

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

2. 新しいコード セルを追加し、それを使用して次のコードを実行します。このコードでは、*Python* を使用して GitHub からボリュームにデータ ファイルをダウンロードします。

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

3. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## ファイル内のデータのクエリを実行する

1. 既存のコード セルの下にマウスを移動し、表示される **[+ コード]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、ファイルからデータを読み込み、最初の 100 行を表示します。

    ```python
    df = spark.read.load(f'/Volumes/{catalog_name}/default/spark_lab/*.csv', format='csv')
    display(df.limit(100))
    ```

1. 出力を確認し、ファイル内のデータが販売注文に関連しているが、列ヘッダーやデータ型に関する情報が含まれていないことに注意してください。 データの意味をよりわかりやすくするために、データフレームの "*スキーマ*" を定義できます。

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。これによりデータのスキーマが定義されます。

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

1. 今回は、データフレームに列ヘッダーが含まれていることを確認します。 次に、新しいコード セルを追加し、それを使用して次のコードを実行することでデータフレーム スキーマの詳細を表示し、正しいデータ型が適用されていることを確認します。

    ```python
   df.printSchema()
    ```

## Spark SQL を使用してデータのクエリを実行する

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```python
   df.createOrReplaceTempView("salesorders")
   spark_df = spark.sql("SELECT * FROM salesorders")
   display(spark_df)
    ```

   前に使用したデータフレーム オブジェクトのネイティブ メソッドを使用すると、データに対するクエリの実行と分析を非常に効率的に行うことができます。 ただし、SQL 構文の方が扱いやすいと考えるデータ アナリストも多くいます。 Spark SQL は、Spark の SQL 言語 API であり、SQL ステートメントの実行だけではなく、リレーショナル テーブル内でのデータの永続化にも使用できます。

   先ほど実行したコードでは、データフレームのデータのリレーショナル "*ビュー*" が作成された後、**spark.sql** ライブラリを使って Python コード内に Spark SQL 構文が埋め込まれ、ビューに対してクエリが実行され、結果がデータフレームとして返されます。

## 結果を視覚化として表示する

1. 新しいコード セルで次のコードを実行して、**salesorders** テーブルに対してクエリを実行します。

    ```sql
   %sql
    
   SELECT * FROM salesorders
    ```

1. 結果の表の上にある **[+]** 、 **[視覚化]** の順に選択して視覚化エディターを表示し、次のオプションを適用します。
    - **視覚化の種類**: 横棒
    - **X 列**: 項目
    - **Y 列**: *新しい列を追加し、***数量**を選択します。 ****合計***集計を適用します*。
    
1. 視覚化を保存し、コード セルを再実行して、結果のグラフをノートブックに表示します。

## matplotlib の使用を開始する

1. 新しいコード セルで、次のコードを実行して、販売注文データをいくつか取得してデータフレームに取り込みます。

    ```python
   sqlQuery = "SELECT CAST(YEAR(OrderDate) AS CHAR(4)) AS OrderYear, \
                   SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue \
            FROM salesorders \
            GROUP BY CAST(YEAR(OrderDate) AS CHAR(4)) \
            ORDER BY OrderYear"
   df_spark = spark.sql(sqlQuery)
   df_spark.show()
    ```

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。これにより **matplotlb** がインポートされ、それを使用してグラフが作成されます。

    ```python
   from matplotlib import pyplot as plt
    
   # matplotlib requires a Pandas dataframe, not a Spark one
   df_sales = df_spark.toPandas()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'])
   # Display the plot
   plt.show()
    ```

1. 結果を確認します。結果には、各年の総収益が縦棒グラフで示されています。 このグラフの生成に使用されているコードの次の機能に注目してください。
    - **matplotlib** ライブラリには Pandas データフレームが必要であるため、Spark SQL クエリによって返される Spark データフレームをこの形式に変換する必要があります。
    - **matplotlib** ライブラリの中核となるのは、**pyplot** オブジェクトです。 これは、ほとんどのプロット機能の基礎となります。

1. 既定の設定では、使用可能なグラフが生成されますが、カスタマイズすべき範囲が大幅に増えます。 新しいコード セルを追加して次のコードを含め、それを実行します。

    ```python
   # Clear the plot area
   plt.clf()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. プロットは、技術的に**図**に含まれています。 前の例では、図が暗黙的に作成されていましたが、明示的に作成することもできます。 新しいセルで次を実行してみます。

    ```python
   # Clear the plot area
   plt.clf()
   # Create a Figure
   fig = plt.figure(figsize=(8,3))
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. 図には複数のサブプロットが含まれており、それぞれに独自の "軸" があります。 このコードを使用して複数のグラフを作成します。

    ```python
   # Clear the plot area
   plt.clf()
   # Create a figure for 2 subplots (1 row, 2 columns)
   fig, ax = plt.subplots(1, 2, figsize = (10,4))
   # Create a bar plot of revenue by year on the first axis
   ax[0].bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   ax[0].set_title('Revenue by Year')
   # Create a pie chart of yearly order counts on the second axis
   yearly_counts = df_sales['OrderYear'].value_counts()
   ax[1].pie(yearly_counts)
   ax[1].set_title('Orders per Year')
   ax[1].legend(yearly_counts.keys().tolist())
   # Add a title to the Figure
   fig.suptitle('Sales Data')
   # Show the figure
   plt.show()
    ```

> **注**: matplotlib を使用したプロットの詳細については、[matplotlib のドキュメント](https://matplotlib.org/)を参照してください。

## seaborn ライブラリを使用する

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。これにより **seaborn** ライブラリ (matplotlib 上に構築され、その複雑さの一部が抽象化されている) を使用してグラフが作成されます。

    ```python
   import seaborn as sns
   
   # Clear the plot area
   plt.clf()
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. **seaborn** ライブラリを使用すると、統計データの複雑なプロットの作成が容易になり、一貫性のあるデータ視覚化を実現するためにビジュアル テーマを制御できます。 新しいセルで次のコードを実行します。

    ```python
   # Clear the plot area
   plt.clf()
   
   # Set the visual theme for seaborn
   sns.set_theme(style="whitegrid")
   
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. matplotlib と同様、 seaborn では複数のグラフの種類がサポートされています。 次のコードを実行して、折れ線グラフを作成します。

    ```python
   # Clear the plot area
   plt.clf()
   
   # Create a bar chart
   ax = sns.lineplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

> **注**: seaborn を使用したプロットの詳細については、[seaborn のドキュメント](https://seaborn.pydata.org/index.html)を参照してください。

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
