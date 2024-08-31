---
lab:
  title: Azure Databricks と Azure OpenAI を使用した LangChain でのマルチステージ推論
---

# Azure Databricks と Azure OpenAI を使用した LangChain でのマルチステージ推論

マルチステージ推論は、AI の最先端のアプローチで、複雑な問題をより小さな管理しやすい段階に分割することを含みます。 ソフトウェア フレームワークである LangChain を利用すると、大規模言語モデル (LLM) を利用するアプリケーションの作成が容易になります。 Azure Databricks と統合する時、LangChain を使用すると、シームレスなデータの読み込み、モデルのラップ、高度な AI エージェントの開発が可能になります。 この組み合わせは、コンテキストの深い理解と複数のステップで推論する能力を必要とする複雑なタスクの処理に、高い効果を発揮します。

このラボは完了するまで、約 **30** 分かかります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure OpenAI リソースをプロビジョニングする

まだ持っていない場合は、Azure サブスクリプションで Azure OpenAI リソースをプロビジョニングします。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure OpenAI** リソースを作成します。
    - **[サブスクリプション]**: "Azure OpenAI Service へのアクセスが承認されている Azure サブスクリプションを選びます"**
    - **[リソース グループ]**: *リソース グループを作成または選択します*
    - **[リージョン]**: *以下のいずれかのリージョンから**ランダム**に選択する*\*
        - オーストラリア東部
        - カナダ東部
        - 米国東部
        - 米国東部 2
        - フランス中部
        - 東日本
        - 米国中北部
        - スウェーデン中部
        - スイス北部
        - 英国南部
    - **[名前]**: "*希望する一意の名前*"
    - **価格レベル**: Standard S0

> \* Azure OpenAI リソースは、リージョンのクォータによって制限されます。 一覧表示されているリージョンには、この演習で使用されるモデル タイプの既定のクォータが含まれています。 リージョンをランダムに選択することで、サブスクリプションを他のユーザーと共有しているシナリオで、1 つのリージョンがクォータ制限に達するリスクが軽減されます。 演習の後半でクォータ制限に達した場合は、別のリージョンに別のリソースを作成する必要が生じる可能性があります。

3. デプロイが完了するまで待ちます。 次に、Azure portal でデプロイされた Azure OpenAI リソースに移動します。

4. 左側のペインで、**[リソース管理]** の下の **[キーとエンドポイント]** を選択します。

5. エンドポイントと使用可能なキーの 1 つをコピーしておきます。この演習で、後でこれを使用します。

## 必要なモジュールをデプロイする

Azure には、モデルのデプロイ、管理、調査に使用できる **Azure AI Studio** という名前の Web ベース ポータルが用意されています。 Azure AI Studio を使用してモデルをデプロイすることで、Azure OpenAI の調査を開始します。

> **注**: Azure AI Studio を使用すると、実行するタスクを提案するメッセージ ボックスが表示される場合があります。 これらを閉じて、この演習の手順に従うことができます。

1. Azure portal にある Azure OpenAI リソースの **[概要]** ページで、**[開始する]** セクションまで下にスクロールし、ボタンを選択して **Azure AI Studio** に移動します。
   
1. Azure AI Studio の左ペインで、**[デプロイ]** ページを選び、既存のモデル デプロイを表示します。 まだデプロイがない場合は、次の設定で **gpt-35-turbo-16k** モデルの新しいデプロイを作成します。
    - **デプロイ名**: *gpt-35-turbo-16k*
    - **モデル**: gpt-35-turbo-16k *(16k モデルが使用できない場合は、gpt-35-turbo を選択し、それに応じてデプロイに名前を付けます)*
    - **モデル バージョン**: *既定のバージョンを使用する*
    - **デプロイの種類**:Standard
    - **1 分あたりのトークンのレート制限**: 5K\*
    - **コンテンツ フィルター**: 既定
    - **動的クォータを有効にする**: 無効
    
1. **[デプロイ]** ページに戻り、次の設定を使用して、**text-embedding-ada-002** モデルの新しいデプロイを作成します。
    - **デプロイ名**: *text-embedding-ada-002*
    - **モデル**: text-embedding-ada-002
    - **モデル バージョン**: *既定のバージョンを使用する*
    - **デプロイの種類**:Standard
    - **1 分あたりのトークンのレート制限**: 5K\*
    - **コンテンツ フィルター**: 既定
    - **動的クォータを有効にする**: 無効

> \* この演習は、1 分あたり 5,000 トークンのレート制限内で余裕を持って完了できます。またこの制限によって、同じサブスクリプションを使用する他のユーザーのために容量を残すこともできます。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure Databricks** リソースを作成します。
    - **サブスクリプション**: *Azure OpenAI リソースの作成に使用したサブスクリプションと同じ Azure サブスクリプションを選択します*
    - **リソース グループ**: *Azure OpenAI リソースを作成したリソース グループと同じです*
    - **リージョン**: *Azure OpenAI リソースを作成したリージョンと同じです*
    - **[名前]**: "*希望する一意の名前*"
    - **価格レベル**: *Premium* または*試用版*

3. **[確認および作成]** を選択し、デプロイが完了するまで待ちます。 次にリソースに移動し、ワークスペースを起動します。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、この手順をスキップし、そのクラスターを使用してこの演習を完了できます。

1. Azure portal で、Azure Databricks ワークスペースが作成されたリソース グループを参照します。
2. Azure Databricks サービス リソースを選択します。
3. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

> **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

4. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。
5. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
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

6. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。

## 必要なライブラリをインストールする

1. Databricks ワークスペースで、**Workspace** セクションに移動します。

2. **[作成]** を選択し、**[ノートブック]** を選択します。

3. ノートブックに名前を付け、言語として [`Python`] を選択します。

4. 最初のコード セルに、次のコードを入力して実行し、必要なライブラリをインストールします。
   
     ```python
    %pip install langchain openai langchain_openai faiss-cpu
     ```

5. インストールが完了したら、新しいセルでカーネルを再起動します。

     ```python
    %restart_python
     ```

6. 新しいセルで、OpenAI モデルの初期化に使用する認証パラメーターを定義し、`your_openai_endpoint` と `your_openai_api_key` を、先ほど OpenAI リソースからコピーしたエンドポイントとキーに置き換えます。

     ```python
    endpoint = "your_openai_endpoint"
    key = "your_openai_api_key"
     ```
     
## ベクトル インデックスを作成し、埋め込みを格納する

ベクトル インデックスは、高次元ベクトル データを効率的に格納および取得できる特殊なデータ構造です。これは、高速類似性検索とニアレストネイバー クエリの実行に不可欠です。 一方、埋め込みは、ベクトル形式でその意味をキャプチャするオブジェクトの数値表現であり、テキストや画像など、さまざまな種類のデータをマシンで処理して理解できます。

1. 新しいセルで、次のコードを実行して、サンプル データセットを読み込みます。

     ```python
    from langchain_core.documents import Document

    documents = [
         Document(page_content="Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="LangChain is a framework designed to simplify the creation of applications using large language models.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="GPT-4 is a powerful language model developed by OpenAI.", metadata={"date_created": "2024-08-22"})
    ]
    ids = ["1", "2", "3"]
     ```
     
1. 新しいセルで、次のコードを実行して、`text-embedding-ada-002` モデルを使用して埋め込みを生成します。

     ```python
    from langchain_openai import AzureOpenAIEmbeddings
     
    embedding_function = AzureOpenAIEmbeddings(
        deployment="text-embedding-ada-002",
        model="text-embedding-ada-002",
        azure_endpoint=endpoint,
        openai_api_key=key,
        chunk_size=1
    )
     ```
     
1. 新しいセルで、次のコードを実行して、ベクトル ディメンションの参照として最初のテキスト サンプルを使用してベクトル インデックスを作成します。

     ```python
    import faiss
      
    index = faiss.IndexFlatL2(len(embedding_function.embed_query("Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.")))
     ```

## レトリバー ベースのチェーンを構築する

レトリバー コンポーネントは、クエリに基づいて関連するドキュメントまたはデータをフェッチします。 これは、取得拡張生成システムなど、分析用に大量のデータの統合を必要とするアプリケーションで特に役立ちます。

1. 新しいセルで、次のコードを実行して、ベクトル インデックスで最も類似したテキストを検索できるレトリバーを作成します。

     ```python
    from langchain.vectorstores import FAISS
    from langchain_core.vectorstores import VectorStoreRetriever
    from langchain_community.docstore.in_memory import InMemoryDocstore

    vector_store = FAISS(
        embedding_function=embedding_function,
        index=index,
        docstore=InMemoryDocstore(),
        index_to_docstore_id={}
    )
    vector_store.add_documents(documents=documents, ids=ids)
    retriever = VectorStoreRetriever(vectorstore=vector_store)
     ```

1. 新しいセルで、次のコードを実行して、レトリバーと `gpt-35-turbo-16k` モデルを使用して QA システムを作成します。
    
     ```python
    from langchain_openai import AzureChatOpenAI
    from langchain_core.prompts import ChatPromptTemplate
    from langchain.chains.combine_documents import create_stuff_documents_chain
    from langchain.chains import create_retrieval_chain
     
    llm = AzureChatOpenAI(
        deployment_name="gpt-35-turbo-16k",
        model_name="gpt-35-turbo-16k",
        azure_endpoint=endpoint,
        api_version="2023-03-15-preview",
        openai_api_key=key,
    )

    system_prompt = (
        "Use the given context to answer the question. "
        "If you don't know the answer, say you don't know. "
        "Use three sentences maximum and keep the answer concise. "
        "Context: {context}"
    )

    prompt1 = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{input}")
    ])

    chain = create_stuff_documents_chain(llm, prompt)

    qa_chain1 = create_retrieval_chain(retriever, chain)
     ```

1. 新しいセルで、次のコードを実行して QA システムをテストします。

     ```python
    result = qa_chain1.invoke({"input": "What is Azure Databricks?"})
    print(result)
     ```

結果の出力には、サンプル データセットに存在する関連ドキュメントに加え、LLM によって生成された生成テキストに基づく回答が表示されます。

## チェーンをマルチチェーン システムに結合する

LangChainは、複数のチェーンを組み合わせてマルチチェーン システムにすることが可能で、言語モデルの機能を強化する汎用性の高いツールです。 このプロセスには、入力を並列または順番に処理できるさまざまなコンポーネントを組み合わせる処理が含まれ、最終的に最後の応答を合成します。

1. 新しいセルで、次のコードを実行して、2 番目のチェーンを作成します。

     ```python
    from langchain_core.prompts import ChatPromptTemplate
    from langchain_core.output_parsers import StrOutputParser

    prompt2 = ChatPromptTemplate.from_template("Create a social media post based on this summary: {summary}")

    qa_chain2 = ({"summary": qa_chain1} | prompt2 | llm | StrOutputParser())
     ```

1. 新しいセルで、次のコードを実行して、指定された入力でマルチステージ チェーンを呼び出します。

     ```python
    result = qa_chain2.invoke({"input": "How can we use LangChain?"})
    print(result)
     ```

最初のチェーンは、提供されたサンプル データセットに基づいて入力に対する回答を提供し、2 番目のチェーンは、最初のチェーンの出力に基づいてソーシャル メディアの投稿を作成します。 この方法を使うと、複数のステップをチェイニングすることで、より複雑なテキスト処理のタスクを処理できます。

## クリーンアップ

Azure OpenAI リソースでの作業が完了したら、**Azure portal** (`https://portal.azure.com`) でデプロイまたはリソース全体を忘れずに削除します。

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
