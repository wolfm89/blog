---
date: "2024-11-28T14:12:14+01:00"
draft: true
title: "Amazon Bedrock Knowledge Base Terraform Module"
tags: ["aws", "bedrock", "rag", "terraform"]

ShareButtons: [linkedin, reddit, x, whatsapp]
showtoc: true
---

## Introduction

### Amazon Bedrock

Since the [public release of ChatGPT](https://openai.com/index/chatgpt/) in November 2022 by OpenAI, the Generative AI (GenAI) space has seen a surge in interest and innovation.
Companies are exploring ways to leverage GenAI to improve customer service, automate repetitive tasks, and enhance user experiences.

Already back in November 2017, Amazon Web Services (AWS) had launched [Amazon SageMaker](https://aws.amazon.com/about-aws/whats-new/2017/11/introducing-amazon-sagemaker/),
a fully managed service that provides every developer and data scientist with the ability to build, train, and deploy machine learning models quickly.
Since then, AWS has continued to expand its machine learning offerings.
Customers use it to bring their own specialized models into production and maintain them throughout their lifecycle.

Training foundation models like GPT-4o though requires significant computational resources and expertise.
To make it easier for developers to leverage existing GenAI models and build applications on top of them,
AWS introduced a new service called [Amazon Bedrock](https://aws.amazon.com/about-aws/whats-new/2023/09/amazon-bedrock-generally-available/) in September 2023.

Amazon Bedrock is a managed service that provides a suite of pre-trained GenAI models, ranging from text-generation models
(Claude by Anthropic, Llama by Meta, Mistral AI, Jamba by AI12, and Titan by Amazon) to embedding models (Amazon Titan Embeddings, Cohere Embed)
and image-generation models (Claude by Anthropic, Titan by Amazon, Stable by Stability AI).
The serverless nature of Bedrock allows developers to focus on building applications without worrying about infrastructure management and the unified API simplifies the integration of GenAI models into applications.

### Customizing Pre-Trained Models

Every use case is unique, and customers often want to customize the pre-trained models to better suit their needs.
Bedrock supports customers by offering two approaches to model customization:

1. **Fine-Tuning**: Customers can [fine-tune the pre-trained models](https://aws.amazon.com/blogs/aws/customize-models-in-amazon-bedrock-with-your-own-data-using-fine-tuning-and-continued-pre-training/) on their own data to adapt them to specific tasks.
2. **Retrieval Augmented Generation (RAG)**: Customers can use the [Retrieval Augmented Generation](https://aws.amazon.com/blogs/machine-learning/build-an-end-to-end-rag-solution-using-knowledge-bases-for-amazon-bedrock-and-aws-cloudformation/) approach to combine the pre-trained models with a retrieval mechanism to generate responses.

Whether fine-tuning or RAG is better suited for a specific use case depends on multiple factors. Fine-tuning excels at creating content aligned with organizational style or compliance but requires expertise and time, while RAG is ideal for dynamic data integration, quick setup, and providing source references, though it may struggle with comprehensive document summarization.

Imagine a customer that wants to integrate order status information into their chatbot. Order status information is constantly changing and requires real-time updates. Fine-tuning a model every time an order status changes is not feasible. In this case, RAG would be a better choice as it can dynamically retrieve the latest order status information from a knowledge base and generate responses.

## RAG with Amazon Bedrock

In this post I'll focus on the RAG approach and show how to set up a knowledge base for Amazon Bedrock using Terraform.
Let's start by quickly summarizing the key components of the RAG approach. The process consists of four key stages: **indexing**, **retrieval**, **augmentation**, and **generation**. It integrates external data sources with user queries to produce contextually grounded and tailored outputs.

- **Indexing**\
   Data is prepared by converting it into vector embeddings, numerical representations used for similarity search. These embeddings, created from unstructured, semi-structured, or structured data, are stored in a vector database for efficient retrieval.
  {{< figure src="images/rag-pre-processing.drawio.png" title="RAG pre-processing" align="center" >}}
- **Retrieval**\
  When a user submits a query, the retriever identifies and selects the most relevant documents from the vector database. This selection is based on similarity measures between the query and the indexed embeddings.
- **Augmentation**\
  The retrieved documents are used to enhance the user's query through prompt engineering, integrating the context directly into the model's input.
- **Generation**\
  The LLM generates a response using the augmented query and retrieved data.

The runtime execution then looks as follows:

{{< figure src="images/rag-runtime-execution.drawio.png" title="RAG runtime execution" align="center" >}}

In AWS Bedrock terms, the RAG approach involves the following components:

1. **Knowledge Base**: The knowledge base instruments the indexing and retrieval stages. It defines the data sources, the embedding model and the vector database.
2. **Embedding Model**: The embedding model converts the data into vector embeddings. AWS supports Amazon Titan Embeddings and Cohere Embed models.
3. **Vector Database**: The vector database stores the embeddings for efficient retrieval. AWS supports Amazon OpenSearch Serverless, Amazon Aurora (RDS), Pinecone, Redis Enterprise Cloud, and MongoDB Atlas.
4. **Data Sources**: The data sources provide the data to be indexed and the strategies (parsing, chunking) for indexing. AWS mainly supports Amazon S3 sources.