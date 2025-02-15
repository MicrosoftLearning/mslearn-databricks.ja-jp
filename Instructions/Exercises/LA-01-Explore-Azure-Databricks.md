---
lab:
  title: Azure Databricks を探索する
---

# Azure Databricks を探索する

Azure Databricks は、一般的なオープンソース Databricks プラットフォームの Microsoft Azure ベースのバージョンです。

Azure Databricks の*ワークスペース*は、Azure 上の Databricks クラスター、データ、およびリソースを管理するための中心点を提供します。

この演習では、Azure Databricks ワークスペースをプロビジョニングし、そのコア機能の一部について説明します。 

この演習の所要時間は約 **20** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure Databricks** リソースを作成します。
    - **[サブスクリプション]**: *Azure サブスクリプションを選択します*
    - **リソース グループ**: *`msl-xxxxxxx` という名前の新しいリソース グループを作成します ("xxxxxxx" は一意の値です)*
    - **リージョン**: *使用可能なリージョンを選択します*
    - **名前**: `databricks-xxxxxxx`*(ここで、"xxxxxxx" は一意の値です)*
    - **価格レベル**: *Premium* または*試用版*

3. **[確認および作成]** を選択し、デプロイが完了するまで待ちます。 次にリソースに移動し、ワークスペースを起動します。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS 以降のランタイム バージョンを持つクラスターが既にある場合は、それを使ってこの演習を完了し、この手順をスキップできます。

1. Azure portal で、**msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) を参照し、Azure Databricks Service リソースを選択します。
1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します (**[その他]** サブメニューを確認する必要がある場合があります)
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

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。

## Spark を使用してデータを分析する

多くの Spark 環境と同様に、Databricks では、ノートブックを使用して、データの探索に使用できるノートと対話型のコード セルを組み合わせることができます。

1. [**products.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv) ファイルを `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv` からローカル コンピューターにダウンロードし、**products.csv** として保存します。
1. サイドバーの **[(+) 新規]** リンク メニューで、**[データを追加またはアップロード]** を選択します。
1. **[テーブルの作成または変更]** を選択し、ダウンロードした **products.csv** ファイルをコンピューターにアップロードします。
1. **[ファイル アップロードからのテーブルの作成または変更]** ページで、ページの右上でクラスターが選択されていることを確認します。 次に、**hive_metastore** カタログとその既定のスキーマを選択して、**products** という名前の新しいテーブルを作成します。
1. **[カタログ エクスプローラー]** ページで、**[製品]** テーブルが作成されたら、**[作成]** ボタン メニューで、**[ノートブック]** を選択してノートブックを作成します。
1. ノートブックで、ノートブックがクラスターに接続されていることを確認し、最初のセルに自動的に追加されたコードを確認します。これは次のようになるはずです。

    ```python
    %sql
    SELECT * FROM `hive_metastore`.`default`.`products`;
    ```

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用してセルを実行し、ダイアログが表示されたらクラスターを起動してアタッチします。
1. コードによって実行される Spark ジョブが完了するまで待ちます。 このコードは、アップロードしたファイルに基づいて作成されたテーブルからデータを取得します。
1. 結果の表の上にある **[+]** 、 **[視覚化]** の順に選択して視覚化エディターを表示し、次のオプションを適用します。
    - **視覚化の種類**: 横棒
    - **X 列**: カテゴリ
    - **Y 列**: "新しい列を追加し、" **ProductID** "を選択します"。** **Count** "集計を適用します"。** **

    視覚化を保存し、次のようにノートブックに表示されることを確認します。

    ![カテゴリ別の製品数を示す横棒グラフ](./images/databricks-chart.png)

## データフレームを使用してデータを分析する

ほとんどのデータ分析では、前の例のような SQL コードの使用で十分ですが、一部のデータ アナリストやデータ サイエンティストは、データを効率的に操作するために、*PySpark* (Python の Spark 最適化バージョン) などのプログラミング言語における*データフレーム*などのネイティブ Spark オブジェクトを使用する可能性があります。

1. ノートブックで、先ほど実行したコード セルのグラフ出力の下で、**[+ コード]** アイコンを使用して新しいセルを追加します。

    > **ヒント**: **[+ コード]** アイコンを表示するには、出力セルの下にマウスを移動することが必要な場合があります。

1. この新しいセルに次のコードを入力して実行します。

    ```python
    df = spark.sql("SELECT * FROM products")
    df = df.filter("Category == 'Road Bikes'")
    display(df)
    ```

1. "ロード バイク" カテゴリの製品を返す新しいセルを実行します。**

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
