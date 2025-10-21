---
lab:
  title: Unity Catalog のメタストアの設定と操作
---

# Unity Catalog のメタストアの設定と操作

Unity Catalog には、Azure Databricks のデータ資産に対応する集中ガバナンス ソリューションが用意されています。 メタストアは Unity Catalog の最上位コンテナーとして機能し、データの分離とアクセス制御の境界を提供する 3 レベルの名前空間を使用してデータ オブジェクトを整理できます。

この演習では、Catalog Explorer を使用してカタログを作成し、SQL コマンドを使用してスキーマとテーブルにデータを設定し、SQL クエリと Catalog Explorer インターフェイスの両方を使用して階層を操作します。

この演習の所要時間は約 **30** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 

十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの「[Unity Catalog とは](https://learn.microsoft.com/azure/databricks/data-governance/unity-catalog/)」の記事を確認してください。

## Azure Databricks ワークスペースを開く

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します。

2. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

3. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

## 既定のカタログを確認する

1. 左側のサイドバー メニューにある **[カタログ]** をクリックし、Catalog Explorer を開きます。

2. 既に作成されているカタログがいくつかあることに注目してください。
   - 組み込みのメタデータを含む**システム** カタログ
   - ワークスペースと同じ名前の既定のカタログ (例: **databricks-* xxxxxxx***)

3. 既定のカタログ (ワークスペースと同じ名前のもの) をクリックして、その構造を確認します。

4. 既定のカタログには、以下の 2 つのスキーマが既に作成されていることに注目してください。
   - **default** - データ オブジェクトを整理するための既定のスキーマ
   - **information_schema** - メタデータを含むシステム スキーマ

5. カタログの詳細パネルで **[詳細]** ボタンをクリックし、**[保存場所]** フィールドと **[種類]** フィールドを確認します。 種類 **MANAGED_CATALOG** は、Databricks がこのカタログ内のデータ資産のストレージとライフサイクルを自動的に管理することを示します。

## 新しいカタログを作成する

既定のカタログを確認したら、データを整理するための独自のカタログを作成します。

1. 左側のメニューから **[カタログ]** を選択します。

2. **[クイック アクセス]** の下にある **[カタログ]** を選択します。
3. **[カタログの作成]** を選択します。
4. **[新しいカタログの作成]** ダイアログで次の操作を行います。
   - **[カタログ名]** を入力します (たとえば、`my_catalog`)
   - カタログの **[種類]** として **[Standard]** を選択します
   - **[保存場所]** には、既定のカタログ名 (例: **databricks-* xxxxxxx***) を選択して、同じ保存場所を使用します。
   - **[作成]** をクリックします。

3. 表示される **[カタログが作成されました]** ウィンドウで、**[カタログの表示]** をクリックします。

4. 新しく作成したカタログをクリックして、その構造を確認します。 先ほど確認したワークスペース カタログと同様に、既定の **default** スキーマと **information_schema** スキーマが含まれていることに注目してください。

## ノートブックを作成する

ノートブックを使用して、新しいカタログ内の Unity Catalog オブジェクトを作成し、確認する SQL コマンドを実行します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
   
2. 既定のノートブック名 (**無題のノートブック *[日付]***) を「`Populate and navigate the metastore`」に変更し、**[接続]** ドロップダウン リストで **[サーバーレス クラスター]** を選択します (まだ選択されていない場合)。 **[サーバーレス]** は既定で有効になっていることに注意してください。

3. ノートブックの最初のセルに次のコードを入力して実行し、新しいカタログを既定値として設定して確認します。

    ```
    %sql
    USE CATALOG <your_catalog_name>;
    SELECT current_catalog();
    ```

## スキーマの作成と管理

1. 新しいセルを追加し、次のコードを実行して、**sales** というスキーマを作成し、それを既定値として設定します。

    ```
    %sql
    CREATE SCHEMA IF NOT EXISTS sales
    COMMENT 'Schema for sales data';

    USE SCHEMA sales;
    SELECT current_schema();
    ```

2. **Catalog Explorer** でカタログに移動して展開し、作成した **sales** スキーマと、**default** および **information_schema** スキーマを確認します。

3. 左側のメニューから **[ワークスペース]** を選択し、ノートブックに移動してノートブックに戻ります。

## テーブルの作成と管理

1. 新しいセルを追加し、次のコードを実行して、顧客データのマネージド テーブルを作成します。

    ```
    %sql
    CREATE OR REPLACE TABLE customers (
      customer_id INT,
      customer_name STRING,
      email STRING,
      city STRING,
      country STRING
    )
    COMMENT 'Customer information table';
    ```

2. 新しいセルを追加し、次のコードを実行してサンプル データを挿入し、挿入されたことを確認します。

    ```
    %sql
    INSERT INTO customers VALUES
      (1, 'Aaron Gonzales', 'aaron@contoso.com', 'Seattle', 'USA'),
      (2, 'Anne Patel', 'anne@contoso.com', 'London', 'UK'),
      (3, 'Casey Jensen', 'casey@contoso.com', 'Toronto', 'Canada'),
      (4, 'Elizabeth Moore', 'elizabeth@contoso.com', 'Sydney', 'Australia'),
      (5, 'Liam Davis', 'liam@contoso.com', 'Berlin', 'Germany');

    SELECT * FROM customers;
    ```
3. **Catalog Explorer** に切り替えて、カタログ > **sales** スキーマ > **customers** テーブルの順に移動します。 テーブルをクリックして次のことを確認します。
   - **[スキーマ]** タブ - 列の定義とデータ型を確認します
   - **[サンプル データ]** タブ - 挿入したデータのプレビューを表示します
   - **[詳細]** タブ - 保存場所やテーブルの種類 (MANAGED) などのテーブルのメタデータを確認します

4. 左側のメニューから **[ワークスペース]** を選択し、ノートブックに移動してノートブックに戻ります。

## ビューを作成して管理する

1. 新しいセルを追加し、次のコードを実行して、顧客をフィルター処理するビューを作成します。

    ```
    %sql
    CREATE OR REPLACE VIEW usa_customers AS
    SELECT customer_id, customer_name, email, city
    FROM customers
    WHERE country = 'USA';

    SELECT * FROM usa_customers;
    ```

2. **Catalog Explorer** に切り替えて、カタログ > **sales** スキーマの順に移動します。 **customers** テーブルと **usa_customers** ビューの両方が一覧表示されていることに注目してください。

3. **usa_customers** ビューをクリックし、**[系列]** タブを選択して、**[系列グラフの表示]** を選択します。 ビューとそのソース テーブル間の依存関係が系列グラフにどのように表示されるかを確認します。 これらのオブジェクトの依存関係は Unity Catalog によって追跡されます。そのため、データ フローを理解し、基となるテーブルへの変更の影響を評価するのに役立ちます。

## SQL コマンドを使用してメタストアを確認する

SQL を使用してオブジェクトを作成し、Catalog Explorer で確認したら、SQL コマンドを使用してメタストア構造をプログラムで調べてみましょう。

1. 左側のメニューから **[ワークスペース]** を選択し、ノートブックに移動してノートブックに戻ります。

2. 新しいセルを追加し、次のコードを実行して、アクセスできるすべてのカタログを一覧表示します。

    ```
    %sql
    SHOW CATALOGS;
    ```

   これにより、**システム** カタログ、ワークスペース カタログ、カスタム カタログなど、アクセスできるすべてのカタログが一覧表示されます。

3. 新しいセルを追加し、次のコードを実行して、現在のカタログ内のすべてのスキーマを一覧表示します。

    ```
    %sql
    SHOW SCHEMAS;
    ```

4. 新しいセルを追加し、次のコードを実行して、現在のスキーマ内のすべてのテーブルとビューを一覧表示します。

    ```
    %sql
    SHOW TABLES;
    ```

5. 新しいセルを追加し、次のコードを実行して、DESCRIBE を使用して詳細なテーブル メタデータを取得します。

    ```
    %sql
    DESCRIBE EXTENDED customers;
    ```

   DESCRIBE EXTENDED コマンドを使用すると、列の定義、テーブルのプロパティ、保存場所など、テーブルに関する包括的な情報を確認できます。

6. 最後にもう一度、**Catalog Explorer** に切り替えます。 ビジュアル インターフェイスに表示される内容と SQL の出力を比較します。 SQL コマンドと Catalog Explorer という両方のアプローチで同じメタデータの異なるビューを利用できるので、Unity Catalog メタストアを柔軟に操作および管理できることに注目してください。

## クリーンアップ

Unity Catalog の確認が終わったら、不要な Azure コストが発生しないように、作成したリソースを削除できます。

1. Azure Databricks ワークスペースのブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. (管理対象リソース グループではなく) Azure Databricks ワークスペースを含むリソース グループを選択します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。 
5. リソース グループ名を入力して、削除することを確認し、**[削除]** を選択します。

    数分後に、リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
