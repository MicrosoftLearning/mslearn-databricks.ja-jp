---
lab:
  title: Azure Databricks で機械学習用にハイパーパラメーターを最適化する
---

# Azure Databricks で機械学習用にハイパーパラメーターを最適化する

この演習では、**Optuna** ライブラリを使用して、Azure Databricks での機械学習モデルのトレーニングのハイパーパラメーターを最適化します。

この演習の所要時間は約 **30** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの「[ハイパーパラメーターのチューニング](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/)」の記事を確認してください。

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
1. 既定のノートブック名 (**無題のノートブック "[日付]"**) を**ハイパーパラメーターのチューニング**に変更し、**[接続]** ドロップダウン リストでクラスターを選びます (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

## データの取り込み

この演習のシナリオは、南極でのペンギンの観察に基づいており、機械学習モデルをトレーニングし、観察されたペンギンの位置情報と体の測定値に基づいて、そのペンギンの種を予測することを目標としています。

> **[引用]**: この演習で使用するペンギンのデータセットは、[Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) と、[Long Term Ecological Research Network](https://lternet.edu/) のメンバーである [Palmer Station, Antarctica LTER](https://pal.lternet.edu/) によって収集されて使用できるようにされているデータのサブセットです。

1. ノートブックの最初のセル内に次のコードを入力します。これは "シェル" コマンドを使用して、GitHub のペンギン データをクラスターで使用されるファイル システムの中にダウンロードします。**

    ```bash
    %sh
    rm -r /dbfs/hyperparam_tune_lab
    mkdir /dbfs/hyperparam_tune_lab
    wget -O /dbfs/hyperparam_tune_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
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
   
   data = spark.read.format("csv").option("header", "true").load("/hyperparam_tune_lab/penguins.csv")
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

最適なハイパーパラメーター値の決定に役立つように、Azure Databricks では [**Optuna**](https://optuna.readthedocs.io/en/stable/index.html) をサポートしています。これは、複数のハイパーパラメーター値を試して、データに最適な組み合わせを見つけることができるライブラリです。

Optuna を使用する際は、まず次のような関数を作成します。

- 関数にパラメーターとして渡される 1 つ以上のハイパーパラメーター値を使ってモデルをトレーニングします。
- "損失" (モデルが完璧な予測パフォーマンスからどれだけ離れているか) を測定するために使用できるパフォーマンス メトリックを計算します**
- さまざまなハイパーパラメーター値を試すことで繰り返し最適化 (最小化) できるように、損失値を返します

1. 新しいセルを追加し、次のコードを使用して関数を作成します。この関数では、ハイパーパラメーターに使用する値の範囲を定義し、ペンギン データを使用して、ペンギンの場所と測定値に基づいてペンギンの種類を予測する分類モデルをトレーニングします。

    ```python
   import optuna
   import mlflow # if you wish to log your experiments
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(trial):
       # Suggest hyperparameter values (maxDepth and maxBins):
       max_depth = trial.suggest_int("MaxDepth", 0, 9)
       max_bins = trial.suggest_categorical("MaxBins", [10, 20, 30])

       # Define pipeline components
       cat_feature = "Island"
       num_features = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=cat_feature, outputCol=cat_feature + "Idx")
       numVector = VectorAssembler(inputCols=num_features, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol=numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=[cat_feature + "Idx", "normalizedFeatures"], outputCol="Features")

       dt = DecisionTreeClassifier(
           labelCol="Species",
           featuresCol="Features",
           maxDepth=max_depth,
           maxBins=max_bins
       )

       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, dt])
       model = pipeline.fit(train)

       # Evaluate the model using accuracy.
       predictions = model.transform(test)
       evaluator = MulticlassClassificationEvaluator(
           labelCol="Species",
           predictionCol="prediction",
           metricName="accuracy"
       )
       accuracy = evaluator.evaluate(predictions)

       # Since Optuna minimizes the objective, return negative accuracy.
       return -accuracy
    ```

1. 新しいセルを追加し、次のコードを使用して最適化実験を実行します。

    ```python
   # Optimization run with 5 trials:
   study = optuna.create_study()
   study.optimize(objective, n_trials=5)

   print("Best param values from the optimization run:")
   print(study.best_params)
    ```

1. このコードは、損失を最小限に抑えるためにトレーニング関数を 5 回繰り返し実行することに注目してください (試行回数は **n_trials** 設定に基づきます)。 各試行は MLflow で記録されているので、**&#9656;** トグルを使用して、コード セル下の **MLflow 実行**の出力を展開し、**実験**のハイパーリンクを選択して結果を確認できます。 各実行にはランダムな名前が割り当てられ、MLflow 実行ビューアーで各実行を表示して、記録されたパラメーターとメトリックの詳細を確認できます。
1. すべての実行が完了すると、見つかった最適なハイパーパラメーター値 (損失が最小となる組み合わせ) の詳細がコードに表示されることを確認します。 この場合、**MaxBins** パラメーターは、3 つの指定できる値 (10、20、30) の一覧からの選択肢として定義されます。最良の値は、一覧内の 0 から始まる項目を示します (つまり、0=10、1=20、2=30)。 **MaxDepth** パラメーターは 0 から 10 のランダムな整数として定義され、最良の結果をもたらした整数値が表示されます。 

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除して、不要な Azure のコストを回避し、サブスクリプションの容量を解放できます。

> **その他の情報**:詳細については、Azure Databricks ドキュメントの「[ハイパーパラメーターのチューニング](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/)」を参照してください。
