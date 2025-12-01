---
lab:
  title: Lakeflow Spark 宣言パイプラインを作成する
---

# Lakeflow Spark 宣言パイプラインを作成する

Lakeflow Spark 宣言型パイプラインは、**宣言型**の方法でデータ パイプラインを構築および実行するための Databricks Lakehouse Platform 内のフレームワークです。 つまり、実現したいデータ変換を指定すると、システムはそれらの変換を効率的に実行する方法を自動的に判断し、従来の Data Engineering の複雑さの多くを処理します。

Lakeflow Spark 宣言型パイプラインは、複雑で具体的な詳細を抽象化することで、ETL (抽出、変換、読み込み) パイプラインの開発を簡略化します。 すべてのステップを指示する手続き型コードを記述する代わりに、SQL または Python で、より単純な宣言型構文を使用します。

このラボは完了するまで、約 **40** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。 Lakeflow Spark 宣言型パイプラインは Databricks の Delta Live Tables (DLT) の進化であり、バッチ ワークロードとストリーミング ワークロードの両方に統一されたアプローチを提供します。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

1. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

1. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

1. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

1. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

1. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 お待ちいただく間に、Azure Databricks ドキュメントの [Lakeflow Spark 宣言型パイプライン](https://learn.microsoft.com/azure/databricks/dlt/)の記事をご確認ください。

## Azure Databricks ワークスペースを開く

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

## ノートブックを作成してデータを取り込む

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。

1. 既定のノートブック名 (**無題のノートブック *[日付]***) を `Data Ingestion and Exploration` に変更し、**[接続]** ドロップダウン リストで、**[サーバーレス SQL ウェアハウス]** がまだ選択されていない場合は選択します。 コンピューティングが実行されていない場合は、開始に 1 分ほどかかる場合があります。

1. ノートブックの最初のセルに次のコードを入力します。このコードにより、Covid データを格納するためのボリュームが作成されます。

     ```sql
    %sql
    CREATE VOLUME IF NOT EXISTS covid_data_volume
     ```

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

1. ノートブックに 2 つ目の (Python) セルを作成し、次のコードを入力します。

    ```python
    import requests

    # Download the CSV file
    url = "https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/covid_data.csv"
    response = requests.get(url)
    response.raise_for_status()

    # Get the current catalog
    catalog_name = spark.sql("SELECT current_catalog()").collect()[0][0]

    # Write directly to Unity Catalog volume
    volume_path = f"/Volumes/{catalog_name}/default/covid_data_volume/covid_data.csv"
    with open(volume_path, "wb") as f:
        f.write(response.content)
    ```

    このコードは、GitHub URL から COVID-19 データを含む CSV ファイルをダウンロードし、現在のカタログ コンテキストを使用して Databricks の Unity Catalog ボリュームに保存します。

1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

## SQL を使用して Lakeflow 宣言型パイプラインを作成する

1. 左側のサイドバーで **[ジョブとパイプライン]** を選択し、**[ETL パイプライン]** を選択します。

1. **[空のファイルで開始する]** を選択します。

1. ダイアログで、最初のファイルの言語として **[SQL]** を選択します。 フォルダー パスを更新する必要はありません。 先に進み、**[選択]** ボタンを選択します。

    ![[コードのフォルダーを選択する] ダイアログのスクリーンショット。](./images/declarative-pipeline-folder.png)

1. パイプラインの名前を `Covid-Pipeline` に変更します。

1. エディターに次のコードを入力します。 必ず**カタログ名をご自分のカタログ名に置き換えてください**。

    ```sql
    CREATE OR REFRESH STREAMING TABLE covid_bronze
    COMMENT "New covid data incrementally ingested from cloud object storage landing zone";

    CREATE FLOW covid_bronze_ingest_flow AS
    INSERT INTO covid_bronze BY NAME
    SELECT 
        Last_Update,
        Country_Region,
        Confirmed,
        Deaths,
        Recovered
    FROM STREAM read_files(
        -- replace with the catalog/schema you are using:
        "/Volumes/<catalog name>/default/covid_data_volume/",
        format => "csv",
        header => true
    );
    ```

    このコードは、Databricks でストリーミング インジェスト パイプラインを設定し、Unity Catalog ボリュームから COVID-19 データを含む新しい CSV ファイルを継続的に読み取り、選択した列を covid_bronze というストリーミング テーブルに挿入し、増分データ処理と分析を可能にします。

1. **[ファイルの実行]** ボタンを選択し、出力を確認します。 エラーが表示された場合は、正しいカタログ名が定義されていることを確認してください。

1. 同じエディターで、次のコードを入力します (前のコードの下に)。

    ```sql
    CREATE OR REFRESH MATERIALIZED VIEW covid_silver(
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
    FROM covid_bronze;
    ```
    このコードは、次の動作により `covid_bronze` ストリーミング テーブルのデータを変換しフィルター処理する `covid_silver` という名前の**マテリアライズド ビュー**を作成または更新します。

    - ✅ `MM/dd/yyyy` 形式を使用して、`Last_Update` 文字列を適切な `Report_Date` に変換する。
    - ✅ ダウンストリーム分析用のキー列 (`Country_Region`、`Confirmed`、`Deaths`、`Recovered`) を選択する。
    - ✅ **データ品質制約**を適用して `Country_Region` が null でないことを確認します。更新中に違反すると、操作は失敗します。
    - 📝 次のビューの目的を説明するコメントを追加する: 分析用に書式設定およびフィルター処理された COVID-19 データ。

    このセットアップは、分析やレポートにクリーンで構造化されたデータの使用を確保するのに役立ちます。

1. **[ファイルの実行]** ボタンを選択し、出力を確認します。 

1. 同じエディターで、次のコードを入力します (前のコードの下に)。

    ```sql
    CREATE OR REFRESH MATERIALIZED VIEW covid_gold
    COMMENT "Aggregated daily data for the US with total counts."
    AS
    SELECT
        Report_Date,
        sum(Confirmed) as Total_Confirmed,
        sum(Deaths) as Total_Deaths,
        sum(Recovered) as Total_Recovered
    FROM covid_silver
    WHERE Country_Region = 'US'
    GROUP BY Report_Date;
    ```

    この SQL コードは、次に示す処理で米国の**日次集計された COVID-19 の統計**を提供する `covid_gold` という名前の**マテリアライズド ビュー**を作成または更新します。

    - 🗓 `Report_Date` でデータをグループ化する
    - 📊 毎日、米国の全地域の `Confirmed`、`Deaths`、`Recovered` のケース数を合計する
    - 💬 次の目的を説明するコメントを追加する: 分析またはレポートのための日次集計の大まかな概要

    この `covid_gold` ビューは、ダッシュボード、レポート、またはデータ サイエンス モデルでの使用に最適化された、medallion アーキテクチャの **"ゴールド レイヤー"** を表します。

1. **[ファイルの実行]** ボタンを選択し、出力を確認します。 

1. **[Catalog エクスプローラー]** に戻ります。 カタログ、既定のスキーマを開き、作成されたさまざまなテーブルとボリュームを調べます。

    ![Lakeflow 宣言型パイプラインを使用してデータを読み込んだ後のカタログ エクスプローラーのスクリーンショット。](./images/declarative-pipeline-catalog-explorer.png)

## 結果を視覚化として表示する

テーブルを作成した後、テーブルをデータフレームに読み込み、データを視覚化することができます。

1. *[データ インジェストと探索]* ノートブックで、新しいコード セルを追加し、次のコードを実行して `covid_gold` をデータフレームに読み込みます。

    ```sql
    %sql
    
    SELECT * FROM covid_gold
    ```

1. 結果の表の上にある **[+]** 、 **[視覚化]** の順に選択して視覚化エディターを表示し、次のオプションを適用します。
    - **視覚化の種類**: 折れ線
    - **X 列**: Report_Date
    - **Y 列**: *新しい列を追加して*、**[Total_Confirmed]** を選択します。 ****合計***集計を適用します*。

1. 視覚化を保存し、結果のグラフをノートブックに表示します。

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
