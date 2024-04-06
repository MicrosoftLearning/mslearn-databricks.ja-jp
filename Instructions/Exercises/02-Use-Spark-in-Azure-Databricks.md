---
lab:
  title: Azure Databricks で Apache Spark を使用する
---

# Azure Databricks で Apache Spark を使用する

Azure Databricks は、一般的なオープンソース Databricks プラットフォームの Microsoft Azure ベースのバージョンです。 Azure Databricks は Apache Spark 上に構築されており、ファイル内のデータの操作を伴うデータ エンジニアリングおよび分析タスクに対して非常にスケーラブルなソリューションを提供します。 Java、Scala、Python、SQL など、幅広いプログラミング言語に対応していることが Spark の利点の 1 つであり、これにより Spark は、データ クレンジングと操作、統計分析と機械学習、データ分析と視覚化など、データ処理ワークロードのソリューションとして高い柔軟性を実現しています。

この演習の所要時間は約 **45** 分です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示されたら、***PowerShell*** 環境を選んで、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの記事「[Azure Databricks での探索的データ分析](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/)」を確認してください。

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

## Spark を使用してデータを調べる

多くの Spark 環境と同様に、Databricks では、ノートブックを使用して、データの探索に使用できるノートと対話型のコード セルを組み合わせることができます。

### ノートブックを作成する

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を "**Spark を使用したデータの探索**" に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

### データの取り込み

1. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

    ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021.csv
    ```

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

### ファイル内のデータのクエリを実行する

1. 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、ファイルからデータを読み込み、最初の 100 行を表示します。

    ```python
   df = spark.read.load('spark_lab/*.csv', format='csv')
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
   df = spark.read.load('/spark_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

1. 今回は、データフレームに列ヘッダーが含まれていることを確認します。 次に、新しいコード セルを追加し、それを使用して次のコードを実行することでデータフレーム スキーマの詳細を表示し、正しいデータ型が適用されていることを確認します。

    ```python
   df.printSchema()
    ```

### データフレームをフィルター処理する

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

### データフレーム内のデータを集計してグループ化する

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

    今回の結果には 1 年あたりの販売注文数が表示されます。 select メソッドには *OrderDate* フィールドの year コンポーネントを抽出するための SQL **year** 関数が含まれており、抽出された year 値に列名を割り当てるために **alias** メソッドが使用されていることに注意してください。 次に、データが派生 *Year* 列によってグループ化され、各グループの行**数**が計算されます。その後、結果として生成されたデータフレームを並べ替えるために、最後に **orderBy** メソッドが使用されます。

> **注**:Azure Databricks での DataFrame の使用について詳しくは、Azure Databricks ドキュメントの [DataFrame の概要 - Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) に関するページを参照してください。

### Spark SQL を使用してデータのクエリを実行する

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```python
   df.createOrReplaceTempView("salesorders")
   spark_df = spark.sql("SELECT * FROM salesorders")
   display(spark_df)
    ```

    前に使用したデータフレーム オブジェクトのネイティブ メソッドを使用すると、データに対するクエリの実行と分析を非常に効率的に行うことができます。 ただし、SQL 構文の方が扱いやすいと考えるデータ アナリストも多くいます。 Spark SQL は、Spark の SQL 言語 API であり、SQL ステートメントの実行だけではなく、リレーショナル テーブル内でのデータの永続化にも使用できます。 先ほど実行したコードでは、データフレームのデータのリレーショナル "*ビュー*" が作成された後、**spark.sql** ライブラリを使って Python コード内に Spark SQL 構文が埋め込まれ、ビューに対してクエリが実行され、結果がデータフレームとして返されます。

### SQL コードをセル内で実行する

1. PySpark コードが含まれているセルに SQL ステートメントを埋め込むことができるのは便利ですが、データ アナリストにとっては、SQL で直接作業できればよいという場合も多くあります。 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```sql
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    次の点に注意してください。
    
    - セルの先頭にある "%sql" 行 (magic と呼ばれます) は、このセル内でこのコードを実行するには、PySpark ではなく、Spark SQL 言語ランタイムを使用する必要があることを示しています。
    - SQL コードにより、前に作成した **salesorder** ビューが参照されます。
    - SQL クエリからの出力は、セルの下に自動的に結果として表示されます。
    
> **注**: Spark SQL とデータフレームの詳細については、[Spark SQL のドキュメント](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html)を参照してください。

## Spark を使用してデータを視覚化する

画像が何千もの言葉を語り、表が何千行にも及ぶデータよりもわかりやすいことは、だれもが知っています。 Azure Databricks のノートブックには、データフレームまたは Spark SQL クエリからのデータ視覚化サポートが含まれていますが、これは包括的なグラフ作成を目的としたものではありません。 ただし、matplotlib や seaborn などの Python グラフィックス ライブラリを使用して、データフレーム内のデータからグラフを作成することができます。

### 結果を視覚化として表示する

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

### matplotlib の使用を開始する

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

### seaborn ライブラリを使用する

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

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。