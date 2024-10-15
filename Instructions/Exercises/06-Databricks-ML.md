---
lab:
  title: '非推奨: Azure Databricks で機械学習を開始する'
---

# Azure Databricks での機械学習の概要

この演習では、Azure Databricks でデータを準備し、機械学習モデルをトレーニングする方法について説明します。

この演習の所要時間は約 **45** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Databricks ドキュメントの記事「[Databricks Machine Learning とは](https://learn.microsoft.com/azure/databricks/machine-learning/)」を確認してください。

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
    - **ノードの種類**: Standard_DS3_v2
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成する

Spark MLLib ライブラリを使って機械学習モデルをトレーニングするコードを実行するので、最初の手順ではワークスペースに新しいノートブックを作成します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を "**機械学習**" に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

## データの取り込み

この演習のシナリオは、南極でのペンギンの観察に基づいており、機械学習モデルをトレーニングし、観察されたペンギンの位置情報と体の測定値に基づいて、そのペンギンの種を予測することを目標としています。

> **[引用]**: この演習で使用するペンギンのデータセットは、[Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) と、[Long Term Ecological Research Network](https://lternet.edu/) のメンバーである [Palmer Station, Antarctica LTER](https://pal.lternet.edu/) によって収集されて使用できるようにされているデータのサブセットです。

1. ノートブックの最初のセル内に次のコードを入力します。これは "シェル" コマンドを使用して、GitHub のペンギン データをクラスターで使用されるファイル システムの中にダウンロードします。**

    ```bash
    %sh
    rm -r /dbfs/ml_lab
    mkdir /dbfs/ml_lab
    wget -O /dbfs/ml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. そのセルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行します。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## データを探索しクリーンアップする
  
データ ファイルを取り込んだので、それをデータフレームに読み込んで表示できます。

1. 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、ファイルからデータを読み込み、それを表示します。

    ```python
   df = spark.read.format("csv").option("header", "true").load("/ml_lab/penguins.csv")
   display(df)
    ```

    このコードによって、データを読み込むために必要な "*Spark ジョブ*" が開始されます。出力は *df* という名前の *pyspark.sql.dataframe.DataFrame* オブジェクトです。 この情報はコードのすぐ下に表示されます。**#9656;** トグルを使用すると、**df: pyspark.sql.dataframe.DataFrame** 出力を展開し、それに含まれる列とそのデータ型の詳細を確認できます。 このデータはテキスト ファイルから読み込まれ、空白の値がいくつか含まれていたため、Spark によってすべての列に**文字列**データ型が割り当てられました。
    
    データ自体は、南極で観察された次のペンギン詳細の測定値で構成されています。
    
    - **Island**: ペンギンが観察された南極の島。
    - **CulmenLength**: ペンギンの嘴峰 (くちばし) の長さ (mm)。
    - **CulmenDepth**: ペンギンの嘴峰 (くちばし) の深さ (mm)。
    - **FlipperLength**: ペンギンのひれ足の長さ (mm)。
    - **BodyMass**: ペンギンの体重 (グラム単位)。
    - **Species**: ペンギンの種を表す整数値。
      - **0**:*アデリーペンギン*
      - **1**:"*ジェンツーペンギン*"
      - **2**:"*ヒゲペンギン*"
      
    このプロジェクトの目的は、ペンギンの種 (機械学習の用語では "*ラベル*" と呼ばれます) を予測するために、ペンギンの観察された特性 (その "*特徴*") を使用することです。
      
    観察結果の中には、一部の特徴に対して *null* または "欠測" データ値が含まれているものがあることに注意してください。 取り込み対象の生のソース データにこのような問題があることは珍しくありません。そのため、機械学習プロジェクトの最初の段階では、通常、データを徹底的に調べてクリーンアップし、機械学習モデルのトレーニングにより適したものにします。
    
1. セルを追加し、それを使用して次のセルを実行し、**dropna** メソッドを使ってし不完全なデータを含む行を削除します。また、**select** メソッドを **col** 関数および **astype** 関数と一緒に使用して、適切なデータ型をデータに適用します。

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = df.dropna().select(col("Island").astype("string"),
                              col("CulmenLength").astype("float"),
                             col("CulmenDepth").astype("float"),
                             col("FlipperLength").astype("float"),
                             col("BodyMass").astype("float"),
                             col("Species").astype("int")
                             )
   display(data)
    ```
    
    ここでも、返されたデータフレーム (今回は *data* という名前) の詳細を切り替えて、データ型が適用されていることを確認できます。また、データをレビューして、不完全なデータを含む行が削除されたことを確認できます。
    
    実際のプロジェクトでは、多くの場合、探索やデータ クレンジングをさらに行って、データのエラーを修正 (または削除) したり、外れ値 (異常に大きな値または小さな値) を特定して削除したり、予測しようとしている各レベルの行数が合理的に同じ数になるようにデータのバランスを取ったりする必要があります。

    > **ヒント**: データフレームで使用できるメソッドと関数の詳細については、[Spark SQL リファレンス](https://spark.apache.org/docs/latest/sql-programming-guide.html)を参照してください。

## データを分割する

この演習では、データが適切にクリーニングされ、機械学習モデルのトレーニングに使用できる状態になっていることを前提とします。 予測しようとしているラベルは、特定のカテゴリ、つまり "*クラス*" (ペンギンの種) です。このため、トレーニングする必要がある機械学習モデルの種類は "*分類*" モデルです。 分類 (および、数値の予測に使用される "*回帰*") は "*教師あり*" 機械学習の 1 つのフォームであり、ここで、予測するラベルの既知の値を含むトレーニング データを使用します。 モデルをトレーニングするプロセスは、実際のところ、特徴量の値と既知のラベル値がどのように相関しているかを計算するために、アルゴリズムをデータに適合させているだけに過ぎません。 その後、トレーニング済みのモデルを、特徴量の値のみがわかっている新しい観察結果に適用し、ラベル値を予測させることができます。

トレーニング済みのモデルの信頼度を確保するための一般的なアプローチは、データの "*一部*" のみを使用してモデルをトレーニングすることです。そして、既知のラベル値を含むデータの一部は、トレーニング済みのモデルをテストし、その予測がどの程度正確かを確認するときに使用できるよう残しておきます。 この目標を達成するために、データセットを 2 つのランダムなサブセットに分割します。 トレーニングにはデータの 70% を使用し、30% をテスト用に残しておきます。

1. 次のコードを使用してコード セルを追加して実行し、データを分割します。

    ```python
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## 特徴エンジニアリングを実行する

生データをクレンジングしたら、通常、データ サイエンティストがモデル トレーニングの準備のための追加作業を行います。 "*特徴エンジニアリング*" として一般に知られるこのプロセスでは、トレーニング データセット内で特徴の最適化を繰り返しながら、最適なモデルを作成します。 具体的に特徴をどのように変更する必要があるかは、データや必要なモデルによって異なりますが、慣れておく必要がある一般的な特徴エンジニアリング タスクはいくつかあります。

### カテゴリ特徴量をエンコードする

機械学習アルゴリズムの基本は通常、特徴とラベルの間の数学的関係を見つけることにあります。 つまり、通常はトレーニング データの特徴を "*数値*" として定義することをお勧めします。 場合によっては、一部の特徴が数値ではなく、文字列で表現された "*カテゴリ*" になっていることがあります。たとえば、このデータセットでは、ペンギンが観察された島の名前がこれに当てはまります。 しかし、アルゴリズムのほとんどで数値の特徴が期待されているため、こうした文字列ベースのカテゴリ値は数値として "*エンコード*" する必要があります。 この場合は、**Spark MLLib** ライブラリの **StringIndexer** を使用し、個別の島の名前ごとに一意の整数インデックスを割り当てることで、島の名前を数値としてエンコードします。

1. 次のコードを実行して、**Island** カテゴリ列の値を数値インデックスとしてエンコードします。

    ```python
   from pyspark.ml.feature import StringIndexer

   indexer = StringIndexer(inputCol="Island", outputCol="IslandIdx")
   indexedData = indexer.fit(train).transform(train).drop("Island")
   display(indexedData)
    ```

    結果として、島名の代わりに、観察が記録された島を表す整数値を含む **IslandIdx** 列が各行に表示されます。

### 数値特徴を正規化 (スケーリング) する

次はデータ内の数値に注目してみましょう。 これらの値 (**CulmenLength**、**CulmenDepth**、**FlipperLength**、**BodyMass**) はすべて、ある種の測定値を表しますが、スケールが異なります。 モデルをトレーニングするとき、測定単位は、さまざまな観察結果の間の相対的な差異ほど重要ではなく、モデル トレーニング アルゴリズムが、より大きな数値で表されている特徴によって左右されることはよくあります。これでは、計算するときの特徴の重要度が偏ってしまいます。 これを軽減するには、一般的には、すべてが同じ相対スケール (たとえば、0.0 から 1.0 の間の 10 進値) になるように、数値特徴量の値を "*正規化*" します。

これを行うために使用するコードは、前に行ったカテゴリ エンコードよりも少し複雑です。 複数の列の値を同時にスケーリングする必要があるため、すべての数値特徴の "*ベクトル*" (基本的には配列) を含む単一の列を作成し、スケーラーを適用して、正規化された同等の値を含む新しいベクトル列を生成する、という手法を使用します。

1. 次のコードを使用して数値特徴を正規化し、正規化前と正規化後のベクトル列の比較を確認してください。

    ```python
   from pyspark.ml.feature import VectorAssembler, MinMaxScaler

   # Create a vector column containing all numeric features
   numericFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   numericColVector = VectorAssembler(inputCols=numericFeatures, outputCol="numericFeatures")
   vectorizedData = numericColVector.transform(indexedData)
   
   # Use a MinMax scaler to normalize the numeric values in the vector
   minMax = MinMaxScaler(inputCol = numericColVector.getOutputCol(), outputCol="normalizedFeatures")
   scaledData = minMax.fit(vectorizedData).transform(vectorizedData)
   
   # Display the data with numeric feature vectors (before and after scaling)
   compareNumerics = scaledData.select("numericFeatures", "normalizedFeatures")
   display(compareNumerics)
    ```

    結果の **numericFeatures** 列には、各行のベクトルが含まれています。 このベクトルには、スケーリングされていない 4 つの数値 (ペンギンの元の測定値) が含まれます。 **#9656 ** トグルを使用すると、不連続値をより明確に表示できます。
    
    **normalizedFeatures** 列にも各ペンギンの観察結果のベクトルが含まれますが、今回のベクトルの値は、各測定値の最小値と最大値に基づいて、相対スケールに正規化されています。

### トレーニング用の特徴とラベルを準備する

次に、すべてをまとめて、すべての特徴 (エンコードされたカテゴリの島名と正規化されたペンギンの測定値) が含まれる列と、モデルをトレーニングして予測したいクラス ラベル (ペンギンの種) が含まれる列を 1 つずつ作成しましょう。

1. 次のコードを実行します。

    ```python
   featVect = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="featuresVector")
   preppedData = featVect.transform(scaledData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   display(preppedData)
    ```

    **特徴**ベクトルには 5 つの値 (エンコードされた島と正規化された嘴峰の長さ、嘴峰の深さ、ひれ足の長さ、体重) が含まれます。 ラベルには、ペンギンの種を示すシンプルな整数コードが含まれます。

## 機械学習モデルのトレーニング

トレーニング データの準備できたので、それを使ってモデルをトレーニングすることができます。 モデルのトレーニングには、特徴とラベルの間の関係を確立しようとする "*アルゴリズム*" を使用できます。 ここでトレーニングしたいのは "*クラス*" のカテゴリを予測するモデルなので、"*分類*" アルゴリズムを使用する必要があります。 分類には多くのアルゴリズムがあります。まずは、よく知られているロジスティック回帰から始めましょう。これは各クラス ラベル値の確率を予測するロジスティック計算中に、特徴量データに適用できる最適な係数を繰り返し見つけようとします。 モデルをトレーニングするには、このロジスティック回帰アルゴリズムをトレーニング データに適合させます。

1. 次のコードを実行してモデルをトレーニングします。

    ```python
   from pyspark.ml.classification import LogisticRegression

   lr = LogisticRegression(labelCol="label", featuresCol="features", maxIter=10, regParam=0.3)
   model = lr.fit(preppedData)
   print ("Model trained!")
    ```

    アルゴリズムのほとんどで、モデルのトレーニング方法を制御するためのパラメーターがサポートされています。 この場合は、ロジスティック回帰アルゴリズムによって、特徴ベクトルを含む列と、既知のラベルを含む列を特定する必要があります。また、このアルゴリズムを使用すると、ロジスティック計算の最適な係数を見つけるために実行される繰り返しの最大回数と、モデルの "*オーバーフィット*" (つまり、トレーニング データではうまく機能するが、新しいデータに適用されたときに適切に一般化されないロジスティクス計算を確立すること) を防ぐために使用される正規化パラメーターを指定することもできます。

## モデルをテストする

トレーニング済みのモデルができました。そのモデルは、残しておいたデータを使ってテストすることができます。 これを行う前に、トレーニング データに適用したものと同じ特徴エンジニアリング変換を、テスト データに対して実行する必要があります (この場合は、島名をエンコードし、測定値を正規化します)。 その後、モデルを使用してテスト データで特徴のラベルを予測し、予測されたラベルと実際の既知のラベルを比較することができます。

1. 次のコードを使用してテスト データを準備し、予測を生成します。

    ```python
   # Prepare the test data
   indexedTestData = indexer.fit(test).transform(test).drop("Island")
   vectorizedTestData = numericColVector.transform(indexedTestData)
   scaledTestData = minMax.fit(vectorizedTestData).transform(vectorizedTestData)
   preppedTestData = featVect.transform(scaledTestData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   
   # Get predictions
   prediction = model.transform(preppedTestData)
   predicted = prediction.select("features", "probability", col("prediction").astype("Int"), col("label").alias("trueLabel"))
   display(predicted)
    ```

    結果には次の列が含まれます。
    
    - **features**: テスト データセットから準備された特徴量データ。
    - **probability**: 各クラスのモデルによって計算される確率。 これは、3 つの確率値を含むベクトルで構成され (3 つのクラスがあるため)、その値の合計は 1.0 になります (ペンギンが 3 つの種のクラスの "*いずれか*" に属する確率が 100% であることを前提としています)。
    - **予測**:予測されたクラス ラベル (確率が最も高いもの)。
    - **trueLabel**: テスト データからの実際の既知のラベル値。
    
    モデルの有効性を評価するには、これらの結果で予測されたラベルと実際のラベルを比較するだけです。 ただし、モデル エバリュエーターを使うことで、より意味のあるメトリックを得ることができます。この場合はマルチクラス分類エバリュエーターを使用します (考えられるクラス ラベルが複数あるため)。

1. 次のコードを使用して、テスト データの結果に基づいて分類モデルの評価メトリックを取得します。

    ```python
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="label", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Individual class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                   evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighted (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

    マルチクラス分類に対して計算される評価メトリックは次のとおりです。
    
    - **精度**: 正しかった全体的な予測の割合。
    - クラスごとのメトリック:
      - **精度**: 正しかったこのクラスの予測の割合。
      - **リコール**: 正しく予測されたこのクラスの実際のインスタンスの割合。
      - **F1 スコア**:精度とリコールの複合メトリック
    - すべてのクラスの精度、リコール、F1 の複合 (重み付けされた) メトリック。
    
    > **注**:モデルの予測パフォーマンスを評価する場合、最初は全体的な精度メトリックを使用するのが最良の方法であるように思えるかもしれません。 しかし、考えてみてください。 ジェンツーペンギンが調査場所のペンギン個体数の 95% を占めるとします。 この場合、常にラベル **1** (ジェンツーペンギンのクラス) を予測するモデルの精度は 0.95 になります。 これは、それが特徴に基づいてペンギンの種を予測する優れたモデルであることを意味するものではありません。 データ サイエンティストが追加メトリックを調べて、考えられる各クラス ラベルに対して分類モデルがどの程度予測できるかを、さらによく理解しようとするのはこのためです。

## パイプラインを使用する

必要な特徴エンジニアリングの手順を実行し、アルゴリズムをデータに適合させることで、モデルをトレーニングしました。 いくつかのテスト データでモデルを使用して予測を生成するには ("*推論*" と呼ばれます)、同じ特徴エンジニアリングの手順をテスト データに適用する必要がありました。 モデルをより効率的に構築し使用するには、データ準備に使用されるトランスフォーマーと、そのトレーニングに使用されるモデルを "*パイプライン*" にカプセル化します。

1. 次のコードを使用して、データ準備とモデルのトレーニング手順をカプセル化するパイプラインを作成します。

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model training algorithm steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=10, regParam=0.3)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    特徴エンジニアリングの手順は、パイプラインによってトレーニングされたモデルにカプセル化されているため、自分で各変換を適用することなく、テスト データでモデルを使用することができます (変換はモデルによって自動的に適用されます)。

1. 次のコードを使用して、パイプラインをテスト データに適用します。

    ```python
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   display(predicted)
    ```

## 別のアルゴリズムを試す

ここまでは、ロジスティック回帰アルゴリズムを使用して分類モデルをトレーニングしました。 パイプライン内のそのステージを変更して、別のアルゴリズムを試してみましょう。

1. 次のコードを実行して、デシジョン ツリー アルゴリズムを使用するパイプラインを作成します。

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = DecisionTreeClassifier(labelCol="Species", featuresCol="Features", maxDepth=10)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    今回のパイプラインでは、特徴準備ステージは以前と同じですが、モデルのトレーニングに "*デシジョン ツリー*" アルゴリズムが使用されています。
    
   1. 次のコードを実行して、テスト データで新しいパイプラインを使用します。

    ```python
   # Get predictions
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   
   # Generate evaluation metrics
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                       evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighed (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

## モデルを保存する

実際には、さまざまなアルゴリズム (およびパラメーター) を使用して、モデルを繰り返しトレーニングしてみることで、データに最適なモデルを見つけます。 ここでは、トレーニングしたデシジョン ツリー モデルを引き続き使用します。 それを保存して、後で新しいペンギン観察結果に使用できるようにしましょう。

1. 次のコードを使用して、モデルを保存します。

    ```python
   model.save("/models/penguin.model")
    ```

    これで現場で新しいペンギンを見つけたときに、保存したモデルを読み込み、それを使用して、その特徴の測定値に基づいてペンギンの種を予測することができます。 モデルを使用して新しいデータから予測を生成することを "*推論*" と言います。

1. 次のコードを使用して、モデルのを読み込み、それを使用して、新しいペンギン観察結果に対して種を予測します。

    ```python
   from pyspark.ml.pipeline import PipelineModel

   persistedModel = PipelineModel.load("/models/penguin.model")
   
   newData = spark.createDataFrame ([{"Island": "Biscoe",
                                     "CulmenLength": 47.6,
                                     "CulmenDepth": 14.5,
                                     "FlipperLength": 215,
                                     "BodyMass": 5400}])
   
   
   predictions = persistedModel.transform(newData)
   display(predictions.select("Island", "CulmenDepth", "CulmenLength", "FlipperLength", "BodyMass", col("prediction").alias("PredictedSpecies")))
    ```

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。

> **その他の情報**:詳細については、[Spark MLLib のドキュメント](https://spark.apache.org/docs/latest/ml-guide.html)を参照してください。
