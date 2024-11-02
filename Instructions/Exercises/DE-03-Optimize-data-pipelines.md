---
lab:
  title: Azure Databricks でパフォーマンスを向上させるためにデータ パイプラインを最適化する
---

# Azure Databricks でパフォーマンスを向上させるためにデータ パイプラインを最適化する

Azure Databricks でデータ パイプラインを最適化すると、パフォーマンスと効率が大幅に向上します。 自動ローダーを使用して、増分データ インジェストを Delta Lake のストレージ レイヤーと組み合わせて使用することで、信頼性と ACID トランザクションが保証されます。 ソルティングを実装するとデータ スキューを防ぐことができますが、Z オーダー クラスタリングでは関連情報を併置することでファイルの読み取りが最適化されます。 Azure Databricks の自動チューニング機能とコストベースのオプティマイザーでは、ワークロードの要件に基づいて設定を調整することで、パフォーマンスをさらに向上させることができます。

このラボは完了するまで、約 **30** 分かかります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

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
    - **ノード タイプ**: Standard_D4ds_v5
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

    > **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成してデータを取り込む

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

2. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-01.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-01.parquet
     ```

3. あたらしいセルで、データセットをデータフレームに読み込むには、次のコードを入力します。
   
     ```python
    # Load the dataset into a DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/yellow_tripdata_2021-01.parquet")
    display(df)
     ```

4. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## 自動ローダーを使用してデータ インジェストを最適化する:

大規模なデータセットを効率的に処理するには、データ インジェストの最適化が不可欠です。 自動ローダーは、クラウド ストレージに到着した新しいデータ ファイルを処理し、さまざまなファイル形式とクラウド ストレージ サービスをサポートするように設計されています。 

自動ローダーは、`cloudFiles` と呼ばれる構造化ストリーミング ソースを提供します。 クラウド ファイル ストレージ上に入力ディレクトリ パスを指定すると、`cloudFiles` ソースでは、新しいファイルが到着したときにそれらが自動的に処理されます。また、そのディレクトリ内の既存のファイルも処理できます。 

1. 新しいセルで、次のコードを実行して、サンプル データを含むフォルダーに基づいてストリームを作成します。

     ```python
     df = (spark.readStream
             .format("cloudFiles")
             .option("cloudFiles.format", "parquet")
             .option("cloudFiles.schemaLocation", "/stream_data/nyc_taxi_trips/schema")
             .load("/nyc_taxi_trips/"))
     df.writeStream.format("delta") \
         .option("checkpointLocation", "/stream_data/nyc_taxi_trips/checkpoints") \
         .option("mergeSchema", "true") \
         .start("/delta/nyc_taxi_trips")
     display(df)
     ```

2. 新しいセルで、次のコードを実行して、新しい Parquet ファイルをストリームに追加します。

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-02_edited.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-02_edited.parquet
     ```
   
新しいファイルには新しい列があるため、ストリームは `UnknownFieldException` エラーで停止します。 ストリームからこのエラーがスローされる前に、自動ローダーによって、データの最新のマイクロバッチに対してスキーマ推論が実行され、新しい列をスキーマの末尾にマージすることによって、スキーマの場所が最新のスキーマで更新されます。 既存の列のデータ型は変更されません。

3. ストリーミング コード セルをもう一度実行し、2 つの新しい列がテーブルに追加されたことを確認します。

   ![新しい列を含む差分テーブル](./images/autoloader-new-columns.png)
   
> 注: `_rescued_data` 列には、型の不一致、大文字と小文字の不一致、またはスキーマに列がないために解析されないデータが含まれています。

4. **[割り込み]** を選択して、データ ストリーミングを停止します。
   
ストリーミング データは差分テーブルに書き込まれます。 Delta Lake には、ACID トランザクション、スキーマの進化、タイム トラベル、ストリーミングとバッチ データ処理の統合など、従来の Parquet ファイルに対する一連の強化機能が用意されているため、Delta Lake は、ビッグ データ ワークロードを管理するための強力なソリューションになります。

## 最適化されたデータ変換

データ スキューは分散コンピューティングにおいて大きな課題であり、特に Apache Spark などのフレームワークを使用したビッグ データ処理において顕著です。 ソルティングはデータ スキューを最適化するための効果的な技術であり、パーティショニングの前にキーにランダムな要素、つまり "ソルト" を追加します。　 このプロセスはデータをより均等にパーティション間で分散させるのに役立ち、よりバランスの取れたワークロードとパフォーマンスの向上をもたらします。

1. 新しいセルで、次のコードを実行して、ランダムな整数を持つ *[salt]* 列を追加して、大きな傾斜したパーティションをより小さなパーティションに分割します。

     ```python
    from pyspark.sql.functions import lit, rand

    # Convert streaming DataFrame back to batch DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/*.parquet")
     
    # Add a salt column
    df_salted = df.withColumn("salt", (rand() * 100).cast("int"))

    # Repartition based on the salted column
    df_salted.repartition("salt").write.format("delta").mode("overwrite").save("/delta/nyc_taxi_trips_salted")

    display(df_salted)
     ```   

## ストレージを最適化する

Delta Lake には、データ ストレージのパフォーマンスと管理を大幅に強化できる最適化コマンドのスイートが用意されています。 `optimize` コマンドは、圧縮や Z オーダーなどの手法を使用してデータをより効率的に整理することで、クエリ速度を向上するように設計されています。

圧縮は、小さなファイルを大きなファイルに統合するもので、特に読み取りクエリにとって有益です。 Z オーダーとは、関連情報が近くに保存されるようにデータポイントを配置することで、クエリ中にこのデータにアクセスする時間を短縮することです。

1. 新しいセルで、次のコードを実行して差分テーブルへの圧縮を実行します。

     ```python
    from delta.tables import DeltaTable

    delta_table = DeltaTable.forPath(spark, "/delta/nyc_taxi_trips")
    delta_table.optimize().executeCompaction()
     ```

2. 新しいセルで、次のコードを実行して Z オーダー クラスタリングを実行します。

     ```python
    delta_table.optimize().executeZOrderBy("tpep_pickup_datetime")
     ```

この手法により、同じファイル セット内に関連情報が併置され、クエリのパフォーマンスが向上します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
