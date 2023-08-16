---
title: "Building a Q&A app capable of answering questions related to your enterprise documents using AWS Bedrock, AWS Kendra, AWS S3 and Streamlit"
date: 2023-08-10T18:57:03+02:00
tags: ["python", "openai", "ai", "aws", "genai", "llm"]
description: "The purpose of this post is to show you how to build another basic Q&A app that is capable of answering questions about your company's internal documents. This time we will use AWS Bedrock, AWS Kendra, AWS S3 and Streamlit to build it."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/building-qa-app-with-aws-bedrock-kendra-s3-and-streamlit).

A couple of months ago, I wrote a post about how to build a quick Q&A app (powered by an LLM) capable of answering questions related to your private documents. You can find the post in [here](https://www.mytechramblings.com/posts/building-a-csharp-enhancing-app-using-openai-gpt4-and-streamlit/).

To build it, we were using the following services:
-  ``Azure OpenAI GPT-4``
-  ``Pinecone``   

Now, I want to built a very similar app, but using **only AWS services** (+ Streamlit for the UI).

But before we start talking tech stuff, I think it’s necessary to explain a couple of concepts to better understand how the app will work.

# **How can an LLM model respond to a question about topics it doesn't know about?**

We have two options for enabling our LLM model to understand and answer our private domain-specific questions:

- **Fine-tune** the LLM on text data covering the topic mentioned.
- Using **Retrieval Augmented Generation (RAG)**, a technique that implements an information retrieval component to the generation process. Allowing us to retrieve relevant information and feed this information into the generation model as a secondary source of information.

We will go with option 2.

The RAG process might sound daunting at first, but it really is not. The process is quite simple once we try to explain it in a simple way, here’s a summary of how it works:
- Run a semantic search against your knowledge database to find content that looks like it could be relevant to the user’s question.
- Construct a prompt consisting of the data extracted from the knowledge database, followed by "Given the above content, answer the following question" and then the user’s question.
- Send the prompt through to an LLM and see what answer comes back.

In the end, all we are doing is extracting data from a database that looks like it could be relevant to the user’s question, constructing a prompt that contains the data from the database and telling the LLM model that it must create the response using only the data present in the prompt.

Here's an example of the prompt:

```text
Answer the following question based on the context below.
If you don't know the answer, just say that you don't know. 
Don't try to make up an answer. Do not answer beyond this context.
---
QUESTION: {User Question}                                            
---
CONTEXT:
{Data extracted from the knowledge database}
```

# **What are knowledge databases?**

With RAG the retrieval of relevant information requires an external "Knowledge database", a place where we can store and use to efficiently retrieve information.    
We can think of this database as the external long-term memory of our LLM.

To retrieve information that is semantically related to our questions, we're going to use a "semantic search database". 

## **Semantic search database**

A semantic search database is a type of database that uses semantic technology to understand the meanings and relationships of words and phrases in order to produce highly relevant search results.

Semantic search is a search technique that uses natural language processing algorithms to understand the meaning and context of words and phrases in order to provide more accurate search results.    

This approach is based on the idea that search engines should not just match keywords in a query, but also try to understand the intent of the user’s search and the relationships between the words used.

Overall, semantic search is designed to provide more precise and meaningful search results that better reflect the user’s intent, rather than just matching keywords. This makes it particularly useful for complex queries, such as those related to scientific research, medical information, or legal documents.


# **Which AWS services are we going to use?**

For the Generative AI LLM models, we will be using:
- [AWS Bedrock](https://aws.amazon.com/bedrock)

For the knowledge database, we will be using:
- [AWS Kendra](https://aws.amazon.com/kendra)
- [AWS S3](https://aws.amazon.com/s3)

The next image shows a more precise diagram of how the AWS services are going to interact between them:

![aws-services-diagram](/img/rag-aws-architecture-diagram.png)

# **How the app works**

- The private documents are being stored in a s3 bucket.
- There is a Kendra index configured with an s3 connector, the index checks the s3 bucket every 5 minutes for new content. If new content is found in the found, then it gets automatically saved into Kendra database.
- When a user runs a query through the ``Streamlit`` app, the app does the following steps:
    - Retrieve the relevant information to the given query from Kendra. 
    - Assembles the prompt. 
    - Sends the prompt to one of the available Bedrock LLM models and get the answer that comes back.

![rag-aws-app-interaction-diagram](/img/rag-aws-app-interaction-diagram.png)

# **Building the app**

There are a few steps required before we can begin developing the app.

## **1. Deploy the required AWS services**

To make the app work we need to deploy the following AWS services:
- An s3 bucket for housing our private docs.
- A Kendra index with an s3 connector.
- An IAM role with the required permissions to make everything work.

You can deploy those resources as you like: using the AWS portal, AWS CDK, Terraform, Pulumi, ...

If you want to take the easy road, I have placed some Terraform files on my [GitHub repository](https://github.com/karlospn/building-qa-app-with-aws-bedrock-kendra-s3-and-streamlit/tree/main/infra) that will create every resource needed on you AWS account.

## **2. Sign up for to the AWS Bedrock preview**

As of today (08/16/2023), AWS Bedrock is still on preview. To access it, you'll need to sign up for the preview.

![rag-aws-bedrock-preview](/img/rag-aws-bedrock-preview.png)

Once admitted to the preview, you will have access only to the Amazon Titan LLM. To utilize any of the third-party LLMs (Anthropic and AI21 Labs LLM models), you must register for access separately.

If you try to use any of this third-party LLMs models using the API before they have give you access, the API will respond with an error message.

From this point forward, I'm going to assume that you have access to AWS Bedrock and all the available LLM models.

## **3. Download the boto3 preview wheels files**

The Q&A app will be a Python-based app, which means that you're going to need the ``boto3`` package.    
``boto3`` is the AWS SDK package for Python, which allow Python apps to make use of AWS services.

Right now (08/16/2023), there is not a global available ``boto3`` package that allows us to work with the AWS Bedrock service. To work with it, you must download the ``boto3`` wheel preview version.

> What is a wheel? Python wheels are a pre-built binary package format for Python modules and libraries. 

If you don't want to lose your time searching for the preview ``boto3`` package, I have  uploaded the ``boto3`` and ``botocore`` wheels that you're going to need to interact with Bedrock on my [GitHub Repository](https://github.com/karlospn/building-qa-app-with-aws-bedrock-kendra-s3-and-streamlit/tree/main).

## **4. Build the Q&A app using LangChain**


# **Testing the query app**
