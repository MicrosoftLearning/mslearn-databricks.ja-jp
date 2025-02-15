---
lab:
  title: Azure Databricks で MLflow を使用する
---

# Azure Databricks で MLflow を使用する

この演習では、MLflow を使用して、Azure Databricks で機械学習モデルのトレーニングと提供を行う方法を確認します。

この演習の所要時間は約 **45** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **注**:この演習では、"モデル提供" をサポートするリージョンに **Premium** Azure Databricks ワークスペースが必要です。** リージョンの Azure Databricks 機能の詳細については、「[Azure Databricks のリージョン](https://learn.microsoft.com/azure/databricks/resources/supported-regions)」を参照してください。 適切なリージョンに *Premium* または "試用版" の Azure Databricks ワークスペースが既にある場合は、この手順をスキップして、既存のワークスペースを使用できます。**

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Databricks ドキュメントの記事「[MLflow ガイド](https://learn.microsoft.com/azure/databricks/mlflow/)」を確認してください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、この手順をスキップし、そのクラスターを使用してこの演習を完了できます。

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
    - **Databricks Runtime のバージョン**: "以下に該当する最新の非ベータ版ランタイム (標準ランタイム バージョン**ではない***) の **<u>ML</u>** エディションを選択します。"
        - "*GPU を使用**しない***"
        - *Scala > **2.11** を含める*
        - "**3.4** 以上の Spark を含む"**
    - **Photon Acceleration を使用する**: <u>オフ</u>にする
    - **ノード タイプ**: Standard_D4ds_v5
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成する

Spark MLLib ライブラリを使って機械学習モデルをトレーニングするコードを実行するので、最初の手順ではワークスペースに新しいノートブックを作成します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック "[日付]"**) を **MLflow** に変更し、**[接続]** ドロップダウン リストでクラスターを選びます (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

## データの取り込みと準備

この演習のシナリオは、南極でのペンギンの観察に基づいており、機械学習モデルをトレーニングして、観察されたペンギンの位置と体の測定値に基づいて種類を予測することを目標としています。

> **[引用]**: この演習で使用するペンギンのデータセットは、[Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) と、[Long Term Ecological Research Network](https://lternet.edu/) のメンバーである [Palmer Station, Antarctica LTER](https://pal.lternet.edu/) によって収集されて使用できるようにされているデータのサブセットです。

1. ノートブックの最初のセル内に次のコードを入力します。これは "シェル" コマンドを使用して、GitHub のペンギン データをクラスターで使用されるファイル システムの中にダウンロードします。**

    ```bash
    %sh
    rm -r /dbfs/mlflow_lab
    mkdir /dbfs/mlflow_lab
    wget -O /dbfs/mlflow_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. そのセルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行します。 その後、コードによって実行される Spark ジョブが完了するまで待ちます。

1. 次に、機械学習用のデータを準備します。 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行します。
    - 不完全な行を削除します
    - 適切なデータ型を適用します
    - データのランダムなサンプルを表示します
    - データを 2 つのデータセットに分割します。1 つはトレーニング用、もう 1 つはテスト用です。


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/mlflow_lab/penguins.csv")
   data = data.dropna().select(col("Island").astype("string"),
                               col("CulmenLength").astype("float"),
                               col("CulmenDepth").astype("float"),
                               col("FlipperLength").astype("float"),
                               col("BodyMass").astype("float"),
                               col("Species").astype("int")
                             )
   display(data.sample(0.2))
   
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## MLflow の実験を実行する

MLflow を使用すると、モデル トレーニング プロセスを追跡して評価メトリックをログに記録する実験を実行できます。 モデル トレーニングの実行の詳細を記録するこの機能は、効果的な機械学習モデルを作成する反復的なプロセスにおいて非常に役立ちます。

モデルのトレーニングと評価に通常使用するのと同じライブラリと手法を使用できますが (この例では Spark MLLib ライブラリを使用します)、それを MLflow 実験のコンテキスト内で行います。これには、プロセス中に重要なメトリックと情報をログに記録する追加のコマンドが含まれます。

1. 新しいセルを追加し、そこに次のコードを入力します。

    ```python
   import mlflow
   import mlflow.spark
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   import time
   
   # Start an MLflow run
   with mlflow.start_run():
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
     
       # parameters
       maxIterations = 5
       regularization = 0.5
   
       # Define the feature engineering and model steps
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
       # Chain the steps as stages in a pipeline
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
       # Log training parameter values
       print ("Training Logistic Regression model...")
       mlflow.log_param('maxIter', algo.getMaxIter())
       mlflow.log_param('regParam', algo.getRegParam())
       model = pipeline.fit(train)
      
       # Evaluate the model and log metrics
       prediction = model.transform(test)
       metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
       for metric in metrics:
           evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
           metricValue = evaluator.evaluate(prediction)
           print("%s: %s" % (metric, metricValue))
           mlflow.log_metric(metric, metricValue)
   
           
       # Log the model itself
       unique_model_name = "classifier-" + str(time.time())
       mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
       modelpath = "/model/%s" % (unique_model_name)
       mlflow.spark.save_model(model, modelpath)
       
       print("Experiment run complete.")
    ```

1. 実験の実行が完了したら、必要に応じて、コード セルの下にある **&#9656;** トグルを使用して、**MLflow 実行**の詳細を展開します。 そこに表示される**実験**のハイパーリンクを使用して、実験の実行が一覧表示された MLflow ページを開きます。 各実行には一意の名前が割り当てられています。
1. 最新の実行を選択し、その詳細を表示します。 セクションを展開すると、ログに記録された**パラメーター**と**メトリック**が表示され、トレーニングされて保存されたモデルの詳細を確認できます。

    > **ヒント**: このノートブックの右側にあるサイドバー メニューの** MLflow 実験**アイコンを使用して、実験の実行の詳細を表示することもできます。

## 関数を作成する

機械学習プロジェクトでは、多くの場合、データ サイエンティストはさまざまなパラメーターを使用してモデルのトレーニングを試み、結果をログに毎回記録します。 これを実現するには、トレーニング プロセスをカプセル化する関数を作成し、試したいパラメーターを使用して呼び出すのが一般的です。

1. 新しいセルで次のコードを実行して、前に使用したトレーニング コードを基にした関数を作成します。

    ```python
   def train_penguin_model(training_data, test_data, maxIterations, regularization):
       import mlflow
       import mlflow.spark
       from pyspark.ml import Pipeline
       from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
       from pyspark.ml.classification import LogisticRegression
       from pyspark.ml.evaluation import MulticlassClassificationEvaluator
       import time
   
       # Start an MLflow run
       with mlflow.start_run():
   
           catFeature = "Island"
           numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
           # Define the feature engineering and model steps
           catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
           numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
           numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
           featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
           algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
           # Chain the steps as stages in a pipeline
           pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
           # Log training parameter values
           print ("Training Logistic Regression model...")
           mlflow.log_param('maxIter', algo.getMaxIter())
           mlflow.log_param('regParam', algo.getRegParam())
           model = pipeline.fit(training_data)
   
           # Evaluate the model and log metrics
           prediction = model.transform(test_data)
           metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
           for metric in metrics:
               evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
               metricValue = evaluator.evaluate(prediction)
               print("%s: %s" % (metric, metricValue))
               mlflow.log_metric(metric, metricValue)
   
   
           # Log the model itself
           unique_model_name = "classifier-" + str(time.time())
           mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
           modelpath = "/model/%s" % (unique_model_name)
           mlflow.spark.save_model(model, modelpath)
   
           print("Experiment run complete.")
    ```

1. 新しいセルで、次のコードを使用してその関数を呼び出します。

    ```python
   train_penguin_model(train, test, 10, 0.2)
    ```

1. 2 回目の実行について、MLflow 実験の詳細を表示します。

## MLflow を使用してモデルを登録してデプロイする

トレーニング実験の実行の詳細を追跡するだけでなく、MLflow を使用して、トレーニングした機械学習モデルを管理できます。 各実験の実行によってトレーニングされたモデルは既にログに記録されています。 モデルを "登録" してデプロイし、クライアント アプリケーションに提供することもできます。**

> **注**:モデル提供は Azure Databricks *Premium* ワークスペースでのみサポートされ、[特定のリージョン](https://learn.microsoft.com/azure/databricks/resources/supported-regions)に限定されています。

1. 最新の実験実行の詳細ページを表示します。
1. **[モデルの登録]** ボタンを使用して、その実験で記録されたモデルを登録し、プロンプトが表示されたら、**Penguin Predictor** という名前の新しいモデルを作成します。
1. モデルが登録されたら、**[モデル]** ページ (左側のナビゲーション バー) を表示し、**Penguin Predictor** モデルを選択します。
1. **Penguin Predictor** モデルのページで、**[推論にモデルを使用する]** ボタンを使用して、次の設定で新しいリアルタイム エンドポイントを作成します。
    - **モデル**:Penguin Predictor
    - **モデルのバージョン**: 1
    - **エンドポイント**: predict-penguin
    - **コンピューティング サイズ**: Small

    サービス エンドポイントは、新しいクラスター内でホストされます。エンドポイントが作成されるまで数分かかる場合があります。
  
1. エンドポイントが作成されたら、右上にある **[エンドポイントのクエリ]** ボタンを使用して、エンドポイントをテストできるインターフェイスを開きます。 次に、テスト インターフェイスの **[ブラウザー]** タブで、次の JSON 要求を入力し、**[要求の送信]** ボタンを使用して、エンドポイントを呼び出し、予測を生成します。

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. ペンギンの特徴にいくつかの異なる値を試してみて、返される結果を観察します。 その後、テスト インターフェイスを閉じます。

## エンティティを削除する

エンドポイントが不要になったら、余計なコストが生じないように削除する必要があります。

**[predict-penguin]** エンドポイント ページの **[&#8285;]** メニューで、**[削除]** を選択します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。

> **その他の情報**:詳細については、[Spark MLLib のドキュメント](https://spark.apache.org/docs/latest/ml-guide.html)を参照してください。
