---
lab:
  title: Azure Databricks で Microsoft Purview と Unity Catalog を使用したデータ プライバシーとガバナンスの実装
---

# Azure Databricks で Microsoft Purview と Unity Catalog を使用したデータ プライバシーとガバナンスの実装

Microsoft Purview を使用すると、データ資産全体にわたる包括的なデータ ガバナンスを実現し、Azure Databricks とシームレスに統合して Lakehouse データを管理し、メタデータをデータ マップに取り込むことができます。 Unity Catalog では、データ管理とガバナンスを一元化することでこれを強化し、Databricks ワークスペース全体のセキュリティとコンプライアンスを簡素化します。

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
    - **ノードの種類**: Standard_DS3_v2
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

    > **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## Unity Catalog を設定する

Unity Catalog メタストアには、セキュリティ保護可能なオブジェクト (テーブル、ボリューム、外部の場所、共有など) とそのオブジェクトへのアクセスを制御するアクセス許可に関するメタデータが登録されます。 各メタストアでは、データを整理できる 3 レベルの名前空間 (`catalog`.`schema`.`table`) が公開されます。 組織が活動しているリージョンごとに、1 つのメタストアが存在する必要があります。 Unity Catalog を操作するには、ユーザーが自分のリージョンのメタストアに接続されているワークスペース上に存在する必要があります。

1. サイドバーで、**カタログ**を選択します。

2. カタログ エクスプローラーには、ワークスペース名を持つ既定の Unity Catalog (**databricks-*xxxxxxx*** (セットアップ スクリプトを使用して作成した場合) が存在する必要があります。 カタログを選択し、右側のウィンドウの上部にある **スキーマの作成**を選択します。

3. 新しいスキーマを **e コマース**と名付け、ワークスペースと共に作成したストレージ ロケーションを選択し、**[作成]** を選びます。

4. カタログを選択し、右側のウィンドウで **[ワークスペース]** タブを選択します。ワークスペースがそれに `Read & Write`アクセスできることを確認します。

## Azure Databricks にサンプル データを取り込む

1. 次のサンプル データ ファイルのダウンロード。
   * [customers.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/customers.csv)
   * [products.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/products.csv)
   * [sales.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/sales.csv)

2. Azure Databricks ワークスペースのカタログ エクスプローラーの上部で**+** を選択し、その後 **[データの追加]** を選択します。

3. 新しいウィンドウで、**[ファイルをボリュームにアップロード]** を選択します。

4. 新しいウィンドウで、`ecommerce`スキーマに移動し、それを展開して **[ボリュームの作成]** を選択します。

5. 新しいボリュームに **sample_data** と名前を付け、 **[作成]** を選択します。

6. 新しいボリュームを選択し、ファイル `customers.csv`、`products.csv`、`sales.csv` をアップロードします。 **[アップロード]** を選択します。

7. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

8. ノートブックの最初のセルに、次のコードを入力し、CSV ファイルからテーブルを作成します。

     ```python
    # Load Customer Data
    customers_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/customers.csv")
    customers_df.write.saveAsTable("ecommerce.customers")

    # Load Sales Data
    sales_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/sales.csv")
    sales_df.write.saveAsTable("ecommerce.sales")

    # Load Product Data
    products_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/products.csv")
    products_df.write.saveAsTable("ecommerce.products")
     ```

>**注:** `.load` ファイル パスで、`databricksxxxxxxx` をカタログ名に置き換えます。

9. カタログ エクスプローラーで、`sample_data` ボリュームに移動し、その中に新しいテーブルがあることを確認します。
    
## Microsoft Purview を設定する

Microsoft Purview は、組織がさまざまな環境でデータを管理およびセキュリティで保護するのに役立つ統合データ ガバナンス サービスです。 データ損失防止、情報保護、コンプライアンス管理などの機能を備えた Microsoft Purview には、そのライフサイクル全体を通じてデータを理解、管理、保護するためのツールが用意されています。

1. [Azure Portal](https://portal.azure.com/) に移動します。

2. **[リソースの作成]** を選択し、**Microsoft Purview** を検索します。

3. 次の設定を使って **Microsoft Purview** リソースを作成します。
    - **[サブスクリプション]**: *Azure サブスクリプションを選択します*
    - **リソース グループ**: * Azure Databricks ワークスペースと同じリソース グループを選択します*
    - **Microsoft Purview アカウント名**: *任意の一意の名前*
    - **位置情報**: * Azure Databricks ワークスペースと同じリージョンを選択します*

4. **[確認および作成]** を選択します。 検証を待ってから、**[作成]** を選択します。

5. デプロイが完了するまで待ちます。 次に、Azure portal でデプロイされた Azure OpenAI リソースに移動します。

6. Microsoft Purview ガバナンス ポータルで、サイドバーの **データマップ** セクションに移動します。

7. **[データソース]** ウィンドウで、**[登録]** を選びます。

8. **[データ ソースの登録]** ウィンドウで、**Azure Databricks** を検索して選択します。 **続行**を選択します。

9. データ ソースに一意の名前を付け、Azure Databricks ワークスペースを選択します。 **登録** を選択します。

## データ プライバシーとガバナンス ポリシーを実装する

1. サイドバーの **データ マップ** セクションで、**[分類]** を選択します。

2. **[分類]** ウィンドウで、**[ + 新規]** を選択し、**PII** (個人を特定できる情報) という名前の新しい分類を作成します。 **[OK]** を選択します。

3. サイドバーで **[Data Catalog]** を選択し、**Customers** テーブルに移動します。

4. PII 分類を電子メールと電話の列に適用します。

5. Azure Databricks に移動し、以前に作成したノートブックを開きます。
 
6. 新しいセルで、次のコードを実行して、PII データへのアクセスを制限するデータ アクセス ポリシーを作成します。

     ```sql
    CREATE OR REPLACE TABLE ecommerce.customers (
      customer_id STRING,
      name STRING,
      email STRING,
      phone STRING,
      address STRING,
      city STRING,
      state STRING,
      zip_code STRING,
      country STRING
    ) TBLPROPERTIES ('data_classification'='PII');

    GRANT SELECT ON TABLE ecommerce.customers TO ROLE data_scientist;
    REVOKE SELECT (email, phone) ON TABLE ecommerce.customers FROM ROLE data_scientist;
     ```

7. data_scientist ロールを持つユーザーとして、顧客テーブルのクエリを試みます。 PII 列 (電子メールと電話) へのアクセスが制限されていることを確認します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
