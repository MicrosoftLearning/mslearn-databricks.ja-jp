---
lab:
  title: Azure Databricks で機械学習用にハイパーパラメーターを最適化する
---

# Azure Databricks で機械学習用にハイパーパラメーターを最適化する

この演習では、**Hyperopt** ライブラリを使って、Azure Databricks での機械学習モデルのトレーニングのハイパーパラメーターを最適化します。

この演習の所要時間は約 **30** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、Azure サブスクリプションに、この演習で必要なコンピューティング コアに対する十分なクォータがあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、ユーザー アカウントが、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可をサブスクリプションに持っていることを前提としています。 クォータやアクセス許可が不十分なためにスクリプトが失敗した場合は、Azure portal で Azure Databricks ワークスペースを対話的に作成してみてください。

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

5. リポジトリを複製した後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの「[ハイパーパラメーターのチューニング](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/)」の記事を確認してください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS **<u>ML</u>** 以降のランタイム バージョンを持つクラスターが既にある場合は、それを使ってこの演習を完了し、この手順をスキップできます。

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します
1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選びます。
1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイドバーで、**[(+) 新規]** タスクを選び、**[クラスター]** を選びます。
1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime のバージョン**:*"以下に該当する最新の非ベータ版ランタイム (標準ランタイム バージョンでは**ない*) の **<u>ML</u>*** エディションを選びます。"*
        - "*GPU を使用**しない***"
        - *Scala > **2.11** を含める*
        - "Spark > **3.4** を含める"**
    - **Photon Acceleration を使用する**:選択を<u>解除</u>する
    - **ノードの種類**: Standard_DS3_v2
    - **アクティビティが** *20* **分ない場合は終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成する

Spark MLLib ライブラリを使って機械学習モデルをトレーニングするコードを実行するので、最初の手順ではワークスペースに新しいノートブックを作成します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック "[日付]"**) を**ハイパーパラメーターのチューニング**に変更し、**[接続]** ドロップダウン リストでクラスターを選びます (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

## データの取り込み

この演習のシナリオは、南極でのペンギンの観察に基づいており、機械学習モデルをトレーニングして、観察されたペンギンの位置と体の測定値に基づいて種類を予測することを目標としています。

> **[引用]**: この演習で使用するペンギンのデータセットは、[Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) と、[Long Term Ecological Research Network](https://lternet.edu/) のメンバーである [Palmer Station, Antarctica LTER](https://pal.lternet.edu/) によって収集されて使用できるようにされているデータのサブセットです。

1. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使って、GitHub からクラスターで使われる Databricks ファイル システム (DBFS) にペンギン データをダウンロードします。**

    ```bash
    %sh
    rm -r /dbfs/hyperopt_lab
    mkdir /dbfs/hyperopt_lab
    wget -O /dbfs/hyperopt_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. 次のセルの右上にある **[&#9656; セルの実行]** メニュー オプションを使って実行します。 その後、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 次に、機械学習用のデータを準備します。 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行します。
    - 不完全な行を削除します
    - 適切なデータ型を適用します
    - データのランダムなサンプルを表示します
    - データを 2 つのデータセットに分割します。1 つはトレーニング用、もう 1 つはテスト用です。


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/hyperopt_lab/penguins.csv")
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

## モデルをトレーニングするためのハイパーパラメーター値を最適化する

最も可能性の高いラベルを計算するアルゴリズムに特徴を当てはめることによって、機械学習モデルをトレーニングします。 アルゴリズムはトレーニング データをパラメーターとして受け取り、特徴とラベルの間の数学的リレーションシップを計算しようとします。 データに加えて、ほとんどのアルゴリズムでは 1 つ以上の "ハイパーパラメーター" を使って、リレーションシップの計算方法に影響を与えます。最適なハイパーパラメーター値を決定することは、反復モデル トレーニング プロセスの重要な部分です。**

最適なハイパーパラメーター値を決定できるように、Azure Databricks には **Hyperopt** のサポートが含まれています。これは、複数のハイパーパラメーター値を試して、データに最適な組み合わせを見つけることができるライブラリです。

Hyperopt を使う最初の手順は、次のような関数を作成することです。

- 関数にパラメーターとして渡される 1 つ以上のハイパーパラメーター値を使ってモデルをトレーニングします。
- "損失" (モデルが完璧な予測パフォーマンスからどれだけ離れているか) を測定するために使用できるパフォーマンス メトリックを計算します**
- さまざまなハイパーパラメーター値を試すことで繰り返し最適化 (最小化) できるように、損失値を返します

1. 新しいセルを追加し、次のコードを利用して、その位置と測定値に基づいてペンギンの種類を予測する分類モデルにペンギン データを使って浴びせかける関数を作成します。

    ```python
   from hyperopt import STATUS_OK
   import mlflow
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(params):
       # Train a model using the provided hyperparameter value
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       mlAlgo = DecisionTreeClassifier(labelCol="Species",    
                                       featuresCol="Features",
                                       maxDepth=params['MaxDepth'], maxBins=params['MaxBins'])
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, mlAlgo])
       model = pipeline.fit(train)
       
       # Evaluate the model to get the target metric
       prediction = model.transform(test)
       eval = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName="accuracy")
       accuracy = eval.evaluate(prediction)
       
       # Hyperopt tries to minimize the objective function, so you must return the negative accuracy.
       return {'loss': -accuracy, 'status': STATUS_OK}
    ```

1. 新しいセルを追加し、以下のコードを使って次を行います。
    - 1 つ以上のハイパーパラメーターに使われる値の範囲を指定する検索空間を定義します (詳細については、Hyperopt ドキュメントの「[Defining a Search Space (検索空間の定義)](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/)」を参照してください)。
    - 使用する Hyperopt アルゴリズムを指定します (詳細については、Hyperopt ドキュメントの「[Algorithms (アルゴリズム)](http://hyperopt.github.io/hyperopt/#algorithms)」を参照してください)。
    - **hyperopt.fmin** 関数を使ってトレーニング関数を繰り返し呼び出し、損失を最小化するようにします。

    ```python
   from hyperopt import fmin, tpe, hp
   
   # Define a search space for two hyperparameters (maxDepth and maxBins)
   search_space = {
       'MaxDepth': hp.randint('MaxDepth', 10),
       'MaxBins': hp.choice('MaxBins', [10, 20, 30])
   }
   
   # Specify an algorithm for the hyperparameter optimization process
   algo=tpe.suggest
   
   # Call the training function iteratively to find the optimal hyperparameter values
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=6)
   
   print("Best param values: ", argmin)
    ```

1. コードがトレーニング関数を (**max_evals** 設定に基づいて) 6 回繰り返し実行していることに注意してください。 各実行は MLflow によって記録され、**&#9656;** トグルを使って、コード セルの下にある **MLflow 実行**の出力を展開し、**実験**のハイパーリンクを選んで表示します。 各実行にはランダムな名前が割り当てられ、MLflow 実行ビューアーで各実行を表示して、記録されたパラメーターとメトリックの詳細を確認できます。
1. すべての実行が完了すると、見つかった最適なハイパーパラメーター値 (損失が最小となる組み合わせ) の詳細がコードに表示されることを確認します。 この場合、**MaxBins** パラメーターは、3 つの指定できる値 (10、20、30) の一覧からの選択肢として定義されます。最良の値は、一覧内の 0 から始まる項目を示します (つまり、0=10、1=20、2=30)。 **MaxDepth** パラメーターは 0 から 10 のランダムな整数として定義され、最良の結果をもたらした整数値が表示されます。 検索空間のハイパーパラメーター値スコープの指定の詳細については、Hyperopt ドキュメントの「[Parameter Expressions (パラメーター式)](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/#parameter-expressions)」を参照してください。

## Trials クラスを使って実行の詳細をログする

MLflow 実験の実行を使って各イテレーションの詳細をログするだけでなく、**hyperopt.Trials** クラスを使って各実行の詳細を記録および表示することもできます。

1. 新しいセルを追加し、次のコードを使って、**Trials** クラスによって記録された各実行の詳細を表示します。

    ```python
   from hyperopt import Trials
   
   # Create a Trials object to track each run
   trial_runs = Trials()
   
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=3,
     trials=trial_runs)
   
   print("Best param values: ", argmin)
   
   # Get details from each trial run
   print ("trials:")
   for trial in trial_runs.trials:
       print ("\n", trial)
    ```

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選び、**[&#9632; 終了]** を選んでシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除して、不要な Azure のコストを回避し、サブスクリプションの容量を解放できます。

> **その他の情報**:詳細については、Azure Databricks ドキュメントの「[ハイパーパラメーターのチューニング](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/)」を参照してください。