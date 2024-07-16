---
layout:     post
rewards: false
title:   LlamaIndex
categories:
    - AI
tags:
   - 大模型


---



# Stages within RAG

There are five key stages within RAG, which in turn will be a part of any larger application you build. These are:

- **Loading**: this refers to getting your data from where it lives – whether it’s text files, PDFs, another website, a database, or an API – into your pipeline. [LlamaHub](https://llamahub.ai/) provides hundreds of connectors to choose from.
- **Indexing**: this means creating a data structure that allows for querying the data. For LLMs this nearly always means creating `vector embeddings`, numerical representations of the meaning of your data, as well as numerous other metadata strategies to make it easy to accurately find contextually relevant data.
- **Storing**: once your data is indexed you will almost always want to store your index, as well as other metadata, to avoid having to re-index it.
- **Querying**: for any given indexing strategy there are many ways you can utilize LLMs and LlamaIndex data structures to query, including sub-queries, multi-step queries and hybrid strategies.
- **Evaluation**: a critical step in any pipeline is checking how effective it is relative to other strategies, or when you make changes. Evaluation provides objective measures of how accurate, faithful and fast your responses to queries are.



# Important concepts within each step

There are also some terms you’ll encounter that refer to steps within each of these stages.

## Loading stage

[**Nodes and Documents**](https://docs.llamaindex.ai/en/stable/module_guides/loading/documents_and_nodes/root.html): A `Document` is a container around any data source - for instance, a PDF, an API output, or retrieve data from a database. A `Node` is the atomic unit of data in LlamaIndex and represents a “chunk” of a source `Document`. Nodes have metadata that relate them to the document they are in and to other nodes.  **Node是Document的chunk**

[**Connectors**](https://docs.llamaindex.ai/en/stable/module_guides/loading/connector/root.html): A data connector (often called a `Reader`) ingests data from different data sources and data formats into `Documents` and `Nodes`.

## Indexing Stage

[**Indexes**](https://docs.llamaindex.ai/en/stable/module_guides/indexing/indexing.html): Once you’ve ingested your data, LlamaIndex will help you index the data into a structure that’s easy to retrieve. This usually involves generating `vector embeddings` which are stored in a specialized database called a `vector store`. Indexes can also store a variety of metadata about your data.

[**Embeddings**](https://docs.llamaindex.ai/en/stable/module_guides/models/embeddings.html) LLMs generate numerical representations of data called `embeddings`. When filtering your data for relevance, LlamaIndex will convert queries into embeddings, and your vector store will find data that is numerically similar to the embedding of your query.

## Querying Stage

[**Retrievers**](https://docs.llamaindex.ai/en/stable/module_guides/querying/retriever/root.html): A retriever defines how to efficiently retrieve relevant context from an index when given a query. Your retrieval strategy is key to the relevancy of the data retrieved and the efficiency with which it’s done.

[**Routers**](https://docs.llamaindex.ai/en/stable/module_guides/querying/router/root.html): A router determines which retriever will be used to retrieve relevant context from the knowledge base. More specifically, the `RouterRetriever` class, is responsible for selecting one or multiple candidate retrievers to execute a query. They use a selector to choose the best option based on each candidate’s metadata and the query.

[**Node Postprocessors**](https://docs.llamaindex.ai/en/stable/module_guides/querying/node_postprocessors/root.html): A node postprocessor takes in a set of retrieved nodes and applies transformations, filtering, or re-ranking logic to them.

[**Response Synthesizers**](https://docs.llamaindex.ai/en/stable/module_guides/querying/response_synthesizers/root.html): A response synthesizer generates a response from an LLM, using a user query and a given set of retrieved text chunks.

## Putting it all together

There are endless use cases for data-backed LLM applications but they can be roughly grouped into three categories:

[**Query Engines**](https://docs.llamaindex.ai/en/stable/module_guides/deploying/query_engine/root.html): A query engine is an end-to-end pipeline that allows you to ask questions over your data. It takes in a natural language query, and returns a response, along with reference context retrieved and passed to the LLM.

[**Chat Engines**](https://docs.llamaindex.ai/en/stable/module_guides/deploying/chat_engines/root.html): A chat engine is an end-to-end pipeline for having a conversation with your data (multiple back-and-forth instead of a single question-and-answer).

[**Agents**](https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/root.html): An agent is an automated decision-maker powered by an LLM that interacts with the world via a set of [tools](https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/tools/llamahub_tools_guide.html). Agents can take an arbitrary number of steps to complete a given task, dynamically deciding on the best course of action rather than following pre-determined steps. This gives it additional flexibility to tackle more complex tasks.



# 高级RAG

- [Advanced Retrieval with LlamaPacks: Elevating RAG in Fewer Lines of Code!](https://freedium.cfd/https://python.plainenglish.io/advanced-retrieval-with-llamapacks-elevating-rag-in-fewer-lines-of-code-5d0497339a3c)
-  [LlamaPacks](https://docs.llamaindex.ai/en/stable/community/llama_packs/root.html)

[**Hybrid Fusion Pack**](https://llamahub.ai/l/llama_packs-fusion_retriever-hybrid_fusion)  依赖QueryFusionRetriever

> 根据输入问题**生成多个查询**，
>
> 然后展开向量+关键词搜索(vector +bm25 search)，
>
> 接着用Reciprocal Rank Fusion(倒数排序融合)对搜索结果进行重排，
>
> 选择topk。

对比 [EnsembleRetriever](https://python.langchain.com/docs/modules/data_connection/retrievers/ensemble)

- Llamalndex的混合搜索多了一个生成多个查询的步骤



[**Query Rewriting Retriever Pack**](https://llamahub.ai/l/llama_packs-fusion_retriever-query_rewrite) 依赖QueryFusionRetriever

> 根据输入问题生成多个查询，默认4个
>
> 然后展开**向量搜索(vectorsearch)，**
>
> 接着用Reciprocal Rank Fusion(倒数排序融合)对搜索结果进行重排，
>
> 选择topk。

对比[rewrite-retrieve-read](https://github.com/langchain-ai/langchain/tree/master/templates/rewrite-retrieve-read)

- rewrite-retrieve-read 只生产一个，然后直接返回搜索结果没有排序





[**Auto Merging Retriever Pack**](https://llamahub.ai/l/llama_packs-auto_merging_retriever)  依赖[AutoMergingRetriever](https://docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever.html#auto-merging-retriever)

> 首先加载文档，建立分层节点图(父节点+子节点)，检索叶子节点，
>
> 然后查看是否已检索到足够多的叶节点，即在**同一个父节点下检索到的叶节点超过给定阈值，**
>
> 会删除检索到的叶节点，它会合并这些叶节点到他们的父节点。（把一些相似的子节点合并到它们的父节点中，减少检索的候选节点的数量，提高检索的效率和准确度。这是基于向量的文档检索的一个常用的技术）

分层节点图

我们将使用`HierarchicalNodeParser`. 这将输出节点的层次结构，从具有较大块大小的顶级节点到具有较小块大小的子节点，其中每个子节点都有一个具有较大块大小的父节点。

默认情况下，层次结构为：

- 第一级：块大小 2048
- 第二级：块大小 512
- 第三级：块大小 128

然后我们将这些节点加载到存储中。叶节点通过向量存储进行索引和检索



[**Small-to-big Retrieval Pack**](https://llamahub.ai/l/llama_packs-recursive_retriever-small_to_big)   依赖RecursiveRetriever

> 对给定输入文档分为一组初始的父分块、将父分块进一步细分为子分块。
>
> 将每个子块链接到其父块，并为子块建立索引。
>
> 检索过程中使用较小的子分块，
>
> 然后将检索到的文本所属的较大文本块提供给大语言模型。

和 [Parent Document Retriever](https://python.langchain.com/docs/modules/data_connection/retrievers/parent_document_retriever) 一样



[**Sentence Window Retriever**](https://llamahub.ai/l/llama_packs-sentence_window_retriever)   依赖RecursiveRetriever

> 基于小到大块检索(Small-to-Big Retrieval)，
>
> 我们可以将文档解析为每个子块中的一个句子，
>
> 然后对单句进行检索，
>
> 并将检索到的句子连同句子窗口(原句两侧的句子)一起传递给 LLM。

