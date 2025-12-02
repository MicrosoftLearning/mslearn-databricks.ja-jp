---
lab:
  title: Azure Databricks で Delta Lake を使用する
---

# Azure Databricks で Delta Lake を使用する

Delta Lake は、データ レイクの上に Spark 用のトランザクション データ ストレージ レイヤーを構築するためのオープンソース プロジェクトです。 Delta Lake では、バッチ データ操作とストリーミング データ操作の両方にリレーショナル セマンティクスのサポートが追加され、Apache Spark を使用して、データ レイク内の基になるファイルに基づくテーブル内のデータを処理しクエリを実行できる *Lakehouse* アーキテクチャを作成できます。

このラボは完了するまで、約 **30** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

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

1. スクリプトが完了するまで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの[Delta Lake の概要](https://docs.microsoft.com/azure/databricks/delta/delta-intro)に関する記事をご確認ください。

## Azure Databricks ワークスペースを開く

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

## ノートブックを作成してデータを取り込む

次に、Spark ノートブックを作成し、この演習で使用するデータをインポートしましょう。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。

2. 既定のノートブック名 (**無題のノートブック *[日付]***) を `Explore Delta Lake` に変更し、**[接続]** ドロップダウン リストで、**[サーバーレス SQL ウェアハウス]** がまだ選択されていない場合は選択します。 コンピューティングが実行されていない場合は、開始に 1 分ほどかかる場合があります。

3. ノートブックの最初のセルに次のコードを入力します。このコードにより、製品データを格納するためのボリュームが作成されます。

     ```sql
    %sql
    CREATE VOLUME IF NOT EXISTS product_data_volume
     ```

4. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。

5. 既存のコード セルの下で、[**+ コード**] アイコンを使用して新しいコード セルを追加します。 次に、新しいセルで、次の Python コードを入力して実行します。

    ```python
    import requests

    # Download the CSV file
    url = "https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv"
    response = requests.get(url)
    response.raise_for_status()

    # Get the current catalog
    catalog_name = spark.sql("SELECT current_catalog()").collect()[0][0]

    # Write directly to Unity Catalog volume
    volume_path = f"/Volumes/{catalog_name}/default/product_data_volume/products.csv"
    with open(volume_path, "wb") as f:
        f.write(response.content)
    ```

    この Python コードは、GitHub URL から製品データを含む CSV ファイルをダウンロードし、現在のカタログ コンテキストを使用してストレージ パスを動的に構築し、Databricks の Unity Catalog ボリュームに直接保存します。

6. 既存のコード セルの下で、[**+ コード**] アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行し、ファイルからデータを読み込み、最初の 10 行を表示します。

    ```python
   df = spark.read.load(volume_path, format='csv', header=True)
   display(df.limit(10))
    ```

## ファイル データをデルタ テーブルに読み込み、更新を実行する

データはデータフレームに読み込まれています。 これを差分テーブルに保持してみましょう。

1. 新しいコード セルを追加し、それを使用して次のコードを実行します。

    ```python
    # Create the table if it does not exist. Otherwise, replace the existing table.
    df.writeTo("Products").createOrReplace()
    ```

    差分レイク テーブルのデータは Parquet 形式で保存されます。 ログ ファイルも作成され、データに加えられた変更が追跡されます。

1. Delta 形式のファイル データは、**DeltaTable** オブジェクトに読み込むことができます。これを使用して、テーブル内のデータを表示および更新できます。 新しいセルで次のコードを実行してデータを更新し、製品 771 の価格を 10% 下げます。

    ```python
    from delta.tables import DeltaTable

    # Reference the Delta table in Unity Catalog
    deltaTable = DeltaTable.forName(spark, "Products")

    # Perform the update
    deltaTable.update(
        condition = "ProductID == 771",
        set = { "ListPrice": "ListPrice * 0.9" })

    # View the updated data as a dataframe
    deltaTable.toDF().show(10)
    ```

    更新は差分フォルダー内のデータに保持され、その場所から読み込まれたすべての新しいデータフレームに反映されます。

## ログと *time-travel* を確認する

データの変更がログされるため、Delta Lake の *time-travel* 機能を使用して、以前のバージョンのデータを表示できます。 

1. 新しいコード セルで次のコードを使用して、製品データの元のバージョンを表示します。

    ```python
    new_df = spark.read.option("versionAsOf", 0).table("Products")
    new_df.show(10)
    ```

1. ログには、データに対する変更の完全な履歴が含まれています。 次のコードを使用して、過去 10 件の変更のレコードを確認します。

    ```python
   deltaTable.history(10).show(10, False, True)
    ```

## テーブル レイアウトを最適化する

テーブル データおよび関連するインデックス データの物理ストレージを再編成して、ストレージ スペースを削減し、テーブルにアクセスするときの入出力効率を向上させることができます。 これは、テーブルに対する大量の挿入、更新、または削除操作の後に特に便利です。

1. 新しいコード セルで、次のコードを使用してレイアウトを最適化し、デルタ テーブル内の古いバージョンのデータ ファイルをクリーンアップします。

     ```python
    spark.sql("OPTIMIZE Products")
    spark.sql("VACUUM Products RETAIN 168 HOURS") # 7 days
     ```

Delta Lake には、危険性のある VACUUM コマンドを実行できないようにする安全性チェックが用意されています。 Databricks **サーバーレス SQL** では、データの破損を防ぐため安全でないファイルの削除が制限されており、サーバーレス環境ではこの安全性チェックをオーバーライドできないため、保持期間を 168 時間未満に設定することはできません。

> **注**: デルタ テーブルに対して VACUUM を実行すると、指定されたデータ保有期間よりも古いバージョンにタイム トラベルできなくなります。

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
