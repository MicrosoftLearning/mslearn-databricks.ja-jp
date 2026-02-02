---
lab:
  title: Azure Databricks で大規模言語モデルを確認する
---

# Azure Databricks で大規模言語モデルを確認する

大規模言語モデル (LLM) は、Azure Databricks および Hugging Face Transformers と統合されている場合に、自然言語処理 (NLP) タスクの強力な資産になる可能性があります。 Azure Databricks は、Hugging Face の広範なライブラリから事前トレーニング済みのモデルを含む、LLM へのアクセス、微調整、デプロイを行うシームレスなプラットフォームを提供します。 モデル推論の場合、Hugging Face のパイプライン クラスは、事前トレーニング済みのモデルの使用を簡略化し、Databricks 環境内で直接さまざまな NLP タスクをサポートします。

このラボは完了するまで、約 **30** 分かかります。

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

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup-serverless.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

     ```powershell
    ./mslearn-databricks/setup-serverless.ps1
     ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。

## ノートブックを作成し、必要なライブラリをインストールする

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します。

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイド バーで、**[(+) 新規]** リンクを使用して、**ノートブック**を作成します。

1. ノートブックに名前を付け、言語として `Python` が選択されていることを確認します。 既定のコンピューティングとして **[サーバーレス]** を選択します。

1. 最初のコード セルに次のコードを入力して実行し、必要なライブラリをインストールします。

    ```python
    %pip install transformers==4.53.0 torch
    dbutils.library.restartPython()
    ```

1. 新しいセルに次のコードを入力して実行し、事前トレーニング済みのモデルを読み込みます。

    ```python
   from transformers import pipeline

   # Load the summarization model with PyTorch weights
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")

   # Load the sentiment analysis model
   sentiment_analyzer = pipeline("sentiment-analysis", model="distilbert/distilbert-base-uncased-finetuned-sst-2-english", revision="714eb0f")

   # Load the translation model
   translator = pipeline("translation_en_to_fr", model="google-t5/t5-base", revision="a9723ea")

   # Load a general purpose model for zero-shot classification and few-shot learning
   classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli", revision="d7645e1") 
    ```

### テキストの要約

要約処理パイプラインでは、長いテキストを簡潔にまとめた要約を生成します。 長さの範囲 (`min_length`、`max_length`) を指定し、サンプリングを使用するかどうか (`do_sample`) を指定することで、生成されるサマリーの正確さまたは創造性を判断できます。 

1. 新しいセルに次のコードを入力します。

     ```python
    text = "Large language models (LLMs) are advanced AI systems capable of understanding and generating human-like text by learning from vast datasets. These models, which include OpenAI's GPT series and Google's BERT, have transformed the field of natural language processing (NLP). They are designed to perform a wide range of tasks, from translation and summarization to question-answering and creative writing. The development of LLMs has been a significant milestone in AI, enabling machines to handle complex language tasks with increasing sophistication. As they evolve, LLMs continue to push the boundaries of what's possible in machine learning and artificial intelligence, offering exciting prospects for the future of technology."
    summary = summarizer(text, max_length=75, min_length=25, do_sample=False)
    print(summary)
     ```

2. セルを実行して、要約されたテキストを表示します。

### センチメントを分析する

感情分析パイプラインは、特定のテキストのセンチメントを決定します。 テキストは、ポジティブ、ネガティブ、ニュートラルなどのカテゴリに分類されます。

1. 新しいセルに次のコードを入力します。

     ```python
    text = "I love using Azure Databricks for NLP tasks!"
    sentiment = sentiment_analyzer(text)
    print(sentiment)
     ```

2. セルを実行して、感情分析の結果を表示します。

### テキストの翻訳

翻訳パイプラインでは、テキストが、ある言語から別の言語に変換されます。 この演習で使用されたタスクは `translation_en_to_fr` で、つまり、特定のテキストを英語からフランス語に翻訳します。

1. 新しいセルに次のコードを入力します。

     ```python
    text = "Hello, how are you?"
    translation = translator(text)
    print(translation)
     ```

2. セルを実行して、翻訳されたテキストをフランス語で表示します。

### テキストを分類する

ゼロショット分類パイプラインを使用すると、モデルはトレーニング中に見られないカテゴリにテキストを分類できます。 そのため、`candidate_labels` パラメータとして事前定義されたラベルが必要です。

1. 新しいセルに次のコードを入力します。

     ```python
    text = "Azure Databricks is a powerful platform for big data analytics."
    labels = ["technology", "health", "finance"]
    classification = classifier(text, candidate_labels=labels)
    print(classification)
     ```

2. セルを実行して、ゼロショット分類の結果を表示します。

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
