---
lab:
  title: AutoML を使用してモデルをトレーニングする
---

# AutoML を使用してモデルをトレーニングする

AutoML は Azure Databricks の機能の 1 つであり、データで複数のアルゴリズムとパラメーターを試行し、最適な機械学習モデルをトレーニングします。

この演習の所要時間は約 **30** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **注**:この演習では、"モデル提供" をサポートするリージョンに **Premium** Azure Databricks ワークスペースが必要です。** リージョンの Azure Databricks 機能の詳細については、「[Azure Databricks のリージョン](https://learn.microsoft.com/azure/databricks/resources/supported-regions)」を参照してください。 適切なリージョンに *Premium* または "試用版" の Azure Databricks ワークスペースが既にある場合は、この手順をスキップして、既存のワークスペースを使用できます。**

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Databricks ドキュメントの記事「[AutoML とは?](https://learn.microsoft.com/azure/databricks/machine-learning/automl/)」を確認してください。

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

## トレーニング データを SQL Warehouse にアップロードする

AutoML を使用して機械学習モデルをトレーニングするには、トレーニング データをアップロードする必要があります。 この演習では、場所や体の測定値などの観察に基づいて、ペンギンを 3 つの種のいずれかに分類するモデルをトレーニングします。 種ラベルを含むトレーニング データを Azure Databricks データ ウェアハウスのテーブルに読み込みます。

1. ワークスペースの Azure Databricks ポータルのサイドバーにある **[SQL]** で、**[SQL Warehouses]** を選択します。
1. ワークスペースに **Starter Warehouse** という名前の SQL ウェアハウスが既に含まれていることを確認します。
1. その SQL ウェアハウスの **[アクション]** ( **&#8285;** ) メニューで、 **[編集]** を選択します。 次に、 **[クラスター サイズ]** プロパティを **[2X-Small]** に設定し、変更を保存します。
1. **[開始]** ボタンを使用して SQL ウェアハウスを起動します (1 - 2 分かかる場合があります)。

> **注**: SQL ウェアハウスの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足している可能性があります。 詳細については、「[必要な Azure vCPU クォータ](https://docs.microsoft.com/azure/databricks/sql/admin/sql-endpoints#required-azure-vcpu-quota)」を参照してください。 このような場合は、ウェアハウスの起動に失敗したときのエラー メッセージで詳しく説明されているように、クォータの引き上げを要求してみてください。 または、このワークスペースを削除し、別のリージョンに新しいワークスペースを作成することもできます。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

1. [**penguins.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv) ファイルを `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv` からローカル コンピューターにダウンロードし、**penguins.csv** として保存します。
1. Azure Databricks ワークスペース ポータルのサイドバーで、**[(+) 新規]** を選択し、**[ファイルのアップロード]** を選択して、コンピューターにダウンロードした **penguins.csv** ファイルをアップロードします。
1. **[データのアップロード]** ページで **既定**のスキーマを選択し、テーブル名を **penguins** に設定します。 次に、ページの左下隅にある **[テーブルの作成]** を選択します。
1. テーブルが作成されたら、その詳細を確認します。

## AutoML 実験を作成する

データを用意したので、それを AutoML で使用してモデルをトレーニングします。

1. 左側のサイドバーで、**[実験]** を選択します。
1. **[実験]** ページで、**[AutoML 実験の作成]** を選択します。
1. 次の設定を使用して AutoML 実験を構成します。
    - **クラスター**:"お使いのクラスターを選択します"**
    - **ML 問題の種類**: 分類
    - **入力トレーニング データセット**: "**既定**のデータベースを参照し、**penguins** テーブルを選択します"**
    - **予測ターゲット**: Species
    - **実験名**: Penguin-classification
    - **詳細構成**:
        - **評価メトリック**: 精度
        - **トレーニング フレームワーク**: lightgbm、sklearn、xgboost
        - **タイムアウト**:5
        - **トレーニング/検証/テストの分割に使用する時間列**: *空白のままにします*
        - **正のラベル**: *空白のままにします*
        - **データの中間保存場所**: MLflow 成果物
1. **[AutoML の開始]** ボタンを使用して、実験を開始します。 表示されているすべての情報ダイアログを閉じます。
1. 実験が完了するまで待ちます。 右側の **[更新]** ボタンを使用すると、生成された実行の詳細を表示できます。
1. 5 分後、実験は終了します。 実行を更新すると、(選択した *[精度]* に基づいて) パフォーマンスが最も高いモデルが一覧の一番上に表示されます。

## パフォーマンスが最も高いモデルをデプロイする

AutoML 実験を実行したら、生成されたパフォーマンスが最も高いモデルを調べることができます。

1. **[Penguin-classification]** 実験ページで **[最適なモデルのノートブックを表示]** を選択して、新しいブラウザー タブで、モデルのトレーニングに使用するノートブックを開きます。
1. ノートブック内のセルをスクロールし、モデルのトレーニングに使用されたコードをメモします。
1. ノートブックが表示されているブラウザー タブを閉じて、**[Penguin-classification]** 実験ページに戻ります。
1. 実行の一覧で、最初の実行 (最適なモデルを生成した実行) の名前を選択して開きます。
1. **[成果物]** セクションで、そのモデルが MLflow 成果物として保存されていることに注意してください。 次に、**[モデルの登録]** ボタンを使用して、そのモデルを **Penguin-Classifier** という名前の新しいモデルとして登録します。
1. 左側のサイドバーで、**[モデル]** ページに切り替えます。 次に、登録したばかりの **Penguin-Classifier** を選択します。
1. **[Penguin-Classifier]** ページで、**[推論にモデルを使用する]** ボタンを使用して、次の設定で新しいリアルタイム エンドポイントを作成します。
    - **モデル**:Penguin-Classifier
    - **モデルのバージョン**: 1
    - **エンドポイント**: classify-penguin
    - **コンピューティング サイズ**:Small

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

**[classify-penguin]** エンドポイント ページの **&#8285;** ページで、**[削除]** を選択します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。

> **その他の情報**:詳細については、Azure Databricks ドキュメントの「[Databricks AutoML のしくみ](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/how-automl-works)」を参照してください。