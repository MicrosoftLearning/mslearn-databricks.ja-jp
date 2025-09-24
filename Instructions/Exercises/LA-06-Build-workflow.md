---
lab:
  title: Azure Databricks Lakeflow ジョブを使用してワークロードをデプロイする
---

# Azure Databricks Lakeflow ジョブを使用してワークロードをデプロイする

Azure Databricks Lakeflow ジョブは、ワークロードを効率的にデプロイするための堅牢なプラットフォームを提供します。 Azure Databricks ジョブや Delta Live Tables などの機能を使用すると、ユーザーは複雑なデータ処理、機械学習、分析パイプラインを調整できます。

このラボは完了するまで、約 **40** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

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

7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 お待ちいただく間に、Azure Databricks ドキュメントの [Lakeflow ジョブ](https://learn.microsoft.com/azure/databricks/jobs/)の記事をご確認ください。

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

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。

2. **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

3. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/workflow_lab
    mkdir /dbfs/workflow_lab
    wget -O /dbfs/workflow_lab/2019.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2019_edited.csv
    wget -O /dbfs/workflow_lab/2020.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2020_edited.csv
    wget -O /dbfs/workflow_lab/2021.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2021_edited.csv
     ```

4. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## ジョブ タスクを作成する

タスクを使用して、データ処理および分析ワークフローを実装します。 ジョブは、1 つ以上のタスクで構成されます。 ノートブック、JARS、Delta Live Tables パイプライン、または Python、Scala、Spark submit、および Java アプリケーションを実行するジョブ タスクを作成できます。 この演習では、視覚化グラフにデータの抽出、変換、読み込みを行うノートブックとしてタスクを作成します。 

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。

2. 既定のノートブック名 (**無題のノートブック *[日付]***) を「`ETL task`」に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

    ノートブックの規定の言語が **Python** に設定されていることを確認します。

3. ノートブックの最初のセルに、データのスキーマを定義し、データセットをデータフレームに読み込む次のコードを入力して実行します。

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
   df = spark.read.load('/workflow_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

4. 既存のコード セルの下で、[**+ コード**] アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、重複する行を削除し、`null` エントリを正しい値に置き換えます。

     ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
     ```
    > **注**: **[税]** 列の値を更新すると、そのデータ型は再び `float` に設定されます。 これは、計算の実行後にデータ型が `double` に変更されるためです。 `double` は `float` よりもメモリ使用量が多いため、列を型キャストして `float` に戻す方がパフォーマンスに優れています。

5. 新しいコード セルで次のコードを実行して、注文データを集計およびグループ化します。

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

## ワークフローを構築する

Azure Databricks が、すべてのジョブのタスク オーケストレーション、クラスター管理、監視、およびエラー レポートを管理します。 ジョブは、すぐに、または使いやすいスケジュール システムを使用して定期的に、または新しいファイルが外部の場所に到着するたびに、またはジョブのインスタンスが常に実行されているように継続的に実行できます。

1. ワークスペースで、![[ワークフロー] アイコン](./images/WorkflowsIcon.svg)をクリックします。 サイドバーの **[ジョブとパイプライン]**。

2. [ジョブとパイプライン] ペインで **[作成]**、**[ジョブ]** の順に選択します。

3. 既定のジョブ名 (**[新しいジョブ *[日付]***) を「`ETL job`」に変更します。

4. 次の設定でジョブを構成します。
    - **タスク名**: `Run ETL task notebook`
    - **種類**: ノートブック
    - **ソース**: ワークスペース
    - **パス**: ETL タスク *ノートブック*を*選択*します。
    - **クラスター**:"お使いのクラスターを選択します"**

5. **[タスクの作成]** を選択します。

6. **[今すぐ実行]** を選択します。

7. ジョブの実行が開始されたら、左サイドバーで **[ジョブ実行]** を選択することで、ジョブの実行を監視できます。

8. ジョブの実行が成功したら、ジョブを選択して出力を確認できます。

さらに、スケジュールに基づくワークフローの実行など、トリガー ベースでジョブを実行することもできます。 定期的なジョブの実行をスケジュールするには、ジョブ タスクを開き、トリガーを追加します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 停止]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
