---
lab:
  title: Lakeflow 宣言型パイプラインを作成する
---

# Lakeflow 宣言型パイプラインを作成する

Lakeflow 宣言型パイプラインは、**宣言型**の方法でデータ パイプラインを構築および実行するための Databricks Lakehouse Platform 内のフレームワークです。 つまり、実現したいデータ変換を指定すると、システムはそれらの変換を効率的に実行する方法を自動的に判断し、従来の Data Engineering の複雑さの多くを処理します。

Lakeflow 宣言型パイプラインは、複雑でローレベルな詳細を抽象化することで、ETL (抽出、変換、読み込み) パイプラインの開発を簡略化します。 すべてのステップを指示する手続き型コードを記述する代わりに、SQL または Python で、より単純な宣言型構文を使用します。

このラボは完了するまで、約 **40** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。 Lakeflow 宣言型パイプラインは Databricks の Delta Live Tables (DLT) の進化であり、バッチ ワークロードとストリーミング ワークロードの両方に統一されたアプローチを提供します。

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

7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 お待ちいただく間に、Azure Databricks ドキュメントの [Lakeflow 宣言型パイプライン](https://learn.microsoft.com/azure/databricks/dlt/)の記事をご確認ください。

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

2. 既定のノートブック名 (**無題のノートブック *[日付]***) を「`Data Ingestion and Exploration`」に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

3. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/covid_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/covid_data.csv
     ```

4. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## SQL を使用して Lakeflow 宣言型パイプラインを作成する

1. 新しいノートブックを作成し、名前を[`Covid Pipeline Notebook`]に変更します。

1. ノートブックの名前の横にある **Python** を選択し、既定の言語を **SQL** に変更します。

1. 最初のセルに次のコードを*実行せずに*入力します。 すべてのセルは、パイプラインの作成後に実行されます。 このコードは、以前ダウンロードした生データによって設定されるマテリアライズド ビューを定義します。

     ```sql
    CREATE OR REFRESH MATERIALIZED VIEW raw_covid_data
    COMMENT "COVID sample dataset. This data was ingested from the COVID-19 Data Repository by the Center for Systems Science and Engineering (CSSE) at Johns Hopkins University."
    AS
    SELECT
      Last_Update,
      Country_Region,
      Confirmed,
      Deaths,
      Recovered
    FROM read_files('dbfs:/delta_lab/covid_data.csv', format => 'csv', header => true)
     ```

1. 最初のセルの下で、**+ コード** アイコンを使用して新しいセルを追加し、分析前に前のテーブルのデータのクエリ、フィルター処理、およびフォーマットを行う次のコードを入力します。

     ```sql
    CREATE OR REFRESH MATERIALIZED VIEW processed_covid_data(
      CONSTRAINT valid_country_region EXPECT (Country_Region IS NOT NULL) ON VIOLATION FAIL UPDATE
    )
    COMMENT "Formatted and filtered data for analysis."
    AS
    SELECT
        TO_DATE(Last_Update, 'MM/dd/yyyy') as Report_Date,
        Country_Region,
        Confirmed,
        Deaths,
        Recovered
    FROM live.raw_covid_data;
     ```

1. 3 番目の新しいコード セルに次のコードを入力します。これにより、パイプラインが正常に実行された後にさらに分析するための強化されたデータ ビューが作成されます。

     ```sql
    CREATE OR REFRESH MATERIALIZED VIEW aggregated_covid_data
    COMMENT "Aggregated daily data for the US with total counts."
    AS
    SELECT
        Report_Date,
        sum(Confirmed) as Total_Confirmed,
        sum(Deaths) as Total_Deaths,
        sum(Recovered) as Total_Recovered
    FROM live.processed_covid_data
    GROUP BY Report_Date;
     ```
     
1. 左側のサイドバーで **[ジョブとパイプライン]** を選択し、**[ETL パイプライン]** を選択します。

1. **[パイプラインの作成**] ページで、次の設定を使用して新しいパイプラインを作成します。
    - **パイプライン名**: `Covid Pipeline`
    - **製品エディション**: 詳細
    - **パイプライン モード**: トリガー
    - **ソース コード**:Users/user@name * フォルダー*で、Covid パイプライン ノートブック* ノートブックを**参照します*。
    - **ストレージ オプション**: Hive メタストア
    - **保存場所**: `dbfs:/pipelines/delta_lab`
    - **ターゲット スキーマ**: *入力する*`default`

1. **[作成]** を選択して、**[開始]** を選択します。 次に、パイプラインが実行されるまで待機します (時間がかかる場合があります)。
 
1. パイプラインが正常に実行された後、最近作成した *[データ インジェストと探索]* ノートブックに戻り、新しいセルで以下のコードを実行して、3 つの新しいテーブルのファイルが指定された保存場所に作成されたことを確認します。

     ```python
    display(dbutils.fs.ls("dbfs:/pipelines/delta_lab/schemas/default/tables"))
     ```

1. 別のコード セルを追加し、次のコードを実行して、テーブルが **デフォルト** データベースに作成されていることを確認します。

     ```sql
    %sql

    SHOW TABLES
     ```

## 結果を視覚化として表示する

テーブルを作成した後、テーブルをデータフレームに読み込み、データを視覚化することができます。

1. *[データ インジェストと探索]* ノートブックで、新しいコード セルを追加し、次のコードを実行して `aggregated_covid_data` をデータフレームに読み込みます。

    ```sql
    %sql
    
    SELECT * FROM aggregated_covid_data
    ```

1. 結果の表の上にある **[+]** 、 **[視覚化]** の順に選択して視覚化エディターを表示し、次のオプションを適用します。
    - **視覚化の種類**: 折れ線
    - **X 列**: Report_Date
    - **Y 列**: *新しい列を追加して*、**[Total_Confirmed]** を選択します。 ****合計***集計を適用します*。

1. 視覚化を保存し、結果のグラフをノートブックに表示します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
