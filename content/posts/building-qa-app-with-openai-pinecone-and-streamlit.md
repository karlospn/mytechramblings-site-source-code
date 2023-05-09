---
title: "Building a GPT-4 Q&A app using Azure OpenAI, Pinecone and Streamlit in just a couple of hours"
date: 2023-05-08T16:53:31+02:00
tags: ["python", "openai", "ai", "azure", "embeddings", "llm"]
description: "This post is going to show you how to build a basic GPT-4 Q&A app capable of answering questions related to your private documents in no more than a couple of hours."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/building-qa-app-with-openai-pinecone-and-streamlit).


I won't tell you anything new if I say that there's currently a boom of topics related to AI. Everyone wants to do some sort of integration with ChatGPT or an LLM model, and companies are frantically searching for use cases on how to integrate GPT into their ecosystem.    
One of the questions I frequently receive nowadays is about using GPT to set up an internal question-answering system for a company.

GPT models have extensive knowledge, but it's obviously generalized knowledge. If you want to ask very specific questions about internal workflows or how your company operates, it's impossible for them to know the answers as they haven't been trained for them.

Setting up an application that uses GPT-4 (the latest version of GPT) and is able to answer internal questions about your company's operations may seem like a complicated task, but the truth is that it's stupidly simple once you understand the three or four most important concepts.    
And that's precisely what I want to show you in this post. It may not be the most complex application out there, but in less than 2 hours, we will build an app that uses GPT-4 to answer internal questions about your company.

But before we start coding, I think it's necessary to explain a couple of concepts to understand what we're going to do.


# **1. How can GPT-4 respond to a question about topics it doesn't know about?**

We have two options for enabling our LLM to understand and answer our private domain-specific questions:

- **Fine-tune** the LLM on text data covering the topic mentioned.
- Using **Retrieval Augmented Generation (RAG)**, a technique that implements an information retrieval component to the generation process. Allowing us to retrieve relevant information and feed this information into the generation model as a secondary source of information.

We will go with option 2.

The implementation might seem daunting at first, but it really is not, here’s a summary of how the entire process will look like:
- Run a semantic search against your knowledge database to find content that looks like it could be relevant to the user’s question.
- Construct a prompt consisting of the data extracted from the knowledge database, followed by "Given the above content, answer the following question" and then the user’s question.
- Send the prompt through to GPT-4 and see what answer comes back.

And that's it, what you're doing is nothing more than extracting the relevant data from a database, construction a prompt like this one

```text
Answer the following question based on the context below.
If you don't know the answer, just say that you don't know. Don't try to make up an answer. Do not answer beyond this context.
---
QUESTION: {User Question}                                            
---
CONTEXT:
{Data extracted from the knowledge database}
"""
```

# **2. Knowledge database and embeddings**

With RAG the retrieval of relevant information requires an external "Knowledge database", a place where we can store and use to efficiently retrieve information. We can think of this as the external long-term memory of our LLM.

We will need to retrieve information that is semantically related to our queries, to do this we need to use "vector embeddings". 

## **2.1. What are embeddings?**

Embeddings are a way to represent words and whole sentences in a numerical manner. We know that computers understand the language of numbers, so we try to encode words in a sentence to numbers such that the computer can read it and process it. 

But reading and processing are not the only things that we want computers to do. We also want computers to build a relationship between each word in a sentence, or document with the other words in the same. 

We want word embeddings to capture the context of the paragraph or previous sentences along with capturing the semantic and syntactic properties and similarities of the same.   
In order to achieve that kind of semantic and syntactic relationship we need to demand more than just mapping a word in a sentence or document to mere numbers. We need a larger representation of those numbers that can represent both semantic and syntactic properties. 

We need vectors embeddings.

To create these vectors embeddings and store it in a knowledge database, we must following the following steps:
- Retrieve a paragraph from the document we want to store in the knowledge database.
- Use the ``text-embedding-ada-002`` LLM model to convert it into a vector embeddings.
- Store the vector embeddings into the knowledge database.

Repeat these 3-step process until the entire set of documents are stored in the knowledge database.


## **2.2. Knowledge database**

Vector databases are used to store and query vectors efficiently. They allow us to search for similar vectors based on their similarity in a high-dimensional space.

We're going to use one of those vector database as our knowledge database, right now there are quite a few options available, but in this post, we will use Pinecone as our vector database.

Why choose Pinecone? Because it is a serverless vector database, which means that there is no maintainance required.


# **3. How the app works**

After breaking down what I believe are the most important concepts on the topic, let's being developing our app. 

To build it, we're going to make use of just 3 technologies:
- Azure OpenAI to interact with the LLM models.
- Pinecone to set up a knowledge database.
- Streamlit to build a simple user interface.

The following diagram shows what we are going to build.

![app-diagram](/img/qa-gpt-app-diagram.png)

To simplify the development of the application, we will divide the responsibility into 2 processes:
- An **ingestion process** that will be responsible for:
    - Reading our private documents (PDFs, words, wiki, JIRA, or whatever kind of document we want to ingest).
    - Converting the docs into vector embeddings using ``Azure OpenAI text-embedding-ada-002`` LLM model.
    - Saving the vector embeddings in Pinecone.
- An **application that allows the user to ask questions about the ingested data**. This application has to:
    - Transform the user's query into a vector embedding using ``Azure OpenAI text-embedding-ada-002`` LLM model.
    - Retrieve the relevant information from PineCone
    - Assemble the prompt
    - Send the prompt through to GPT-4 and get the answer that comes back.


# **4. Building the ingest process**

## **4.1. Creating the Pinecone database**

![pinecone-creation](/img/qa-gpt-app-pinecone-index-creation.png)


The embedding model we will use to create the vectors, text-embedding-ada-002, outputs vectors with 1536 dimensions. 

![pinecone-ready](/img/qa-gpt-app-pinecone-index.png)


## **4.2. Deploying a text-embedding-ada-002 model on Azure OpenAI**

![model-deployment](/img/qa-gpt-app-openai-deployments.png)


## **4.3. Building the ingest data process using LangChain**

```python
```

# **5. Building the query app**

## **5.1. Deploying a GPT-4 model on Azure OpenAI**

## **5.2. Building the querying app using Streamlit**

```python
import streamlit as st
import openai
import pinecone
import os


# Check if environment variables are present. If not, throw an error
if os.getenv('PINECONE_API_KEY') is None:
    st.error("PINECONE_API_KEY not set. Please set this environment variable and restart the app.")
if os.getenv('PINECONE_ENVIRONMENT') is None:
    st.error("PINECONE_ENVIRONMENT not set. Please set this environment variable and restart the app.")
if os.getenv('PINECONE_INDEX') is None:
    st.error("PINECONE_INDEX not set. Please set this environment variable and restart the app.")
if os.getenv('AZURE_OPENAI_APIKEY') is None:
    st.error("AZURE_OPENAI_APIKEY not set. Please set this environment variable and restart the app.")
if os.getenv('AZURE_OPENAI_BASE_URI') is None:
    st.error("AZURE_OPENAI_BASE_URI not set. Please set this environment variable and restart the app.")
if os.getenv('AZURE_OPENAI_EMBEDDINGS_MODEL_NAME') is None:
    st.error("AZURE_OPENAI_EMBEDDINGS_MODEL_NAME not set. Please set this environment variable and restart the app.")
if os.getenv('AZURE_OPENAI_GPT4_MODEL_NAME') is None:
    st.error("AZURE_OPENAI_GPT4_MODEL_NAME not set. Please set this environment variable and restart the app.")


st.title("Q&A App 🔎")
query = st.text_input("What do you want to know?")

if st.button("Search"):
   
    # # get Pinecone API environment variables
    pinecone_api = os.getenv('PINECONE_API_KEY')
    pinecone_env = os.getenv('PINECONE_ENVIRONMENT')
    pinecone_index = os.getenv('PINECONE_INDEX')
    
    # # get Azure OpenAI environment variables
    openai.api_key = os.getenv('AZURE_OPENAI_APIKEY')
    openai.api_base = os.getenv('AZURE_OPENAI_BASE_URI')
    openai.api_type = 'azure'
    openai.api_version = '2023-03-15-preview'
    embeddings_model_name = os.getenv('AZURE_OPENAI_EMBEDDINGS_MODEL_NAME')
    gpt4_model_name = os.getenv('AZURE_OPENAI_GPT4_MODEL_NAME')
 
    # Initialize Pinecone client and set index
    pinecone.init(api_key=pinecone_api, environment=pinecone_env)
    index = pinecone.Index(pinecone_index)
 
    # Convert your query into a vector using Azure OpenAI
    try:
        query_vector = openai.Embedding.create(
            input=query,
            engine=embeddings_model_name,
        )["data"][0]["embedding"]
    except Exception as e:
        st.error(f"Error calling OpenAI Embedding API: {e}")
        st.stop()
 
    # Search for the most similar vectors in Pinecone
    search_response = index.query(
        top_k=3,
        vector=query_vector,
        include_metadata=True)

    chunks = [item["metadata"]['text'] for item in search_response['matches']]
 
    # Combine texts into a single chunk to insert in the prompt
    joined_chunks = "\n".join(chunks)

    # Write the selected chunks into the UI
    with st.expander("Chunks"):
        for i, t in enumerate(chunks):
            t = t.replace("\n", " ")
            st.write("Chunk ", i, " - ", t)
    
    with st.spinner("Summarizing..."):
        try:
            # Build the prompt
            prompt = f"""
            Answer the following question based on the context below.
            If you don't know the answer, just say that you don't know. Don't try to make up an answer. Do not answer beyond this context.
            ---
            QUESTION: {query}                                            
            ---
            CONTEXT:
            {joined_chunks}
            """
 
            # Run chat completion using GPT-4
            response = openai.ChatCompletion.create(
                engine=gpt4_model_name,
                messages=[
                    { "role": "system", "content":  "You are a Q&A assistant." },
                    { "role": "user", "content": prompt }
                ],
                temperature=0.7,
                max_tokens=1000
            )
 
            # Write query answer
            st.markdown("### Answer:")
            st.write(response.choices[0]['message']['content'])
   
   
        except Exception as e:
            st.error(f"Error with OpenAI Chat Completion: {e}")
```

# **6. Testing the app**

The last step is to test that the query app works correctly.

- Question 1:

![app-result-1](/img/qa-gpt-app-streamlit-result-1.png)

For every question we can inspect whichs chunks are being retrieved from Pinecone and feed to GPT-4 to build the response

![app-result-chunk](/img/qa-gpt-app-streamlit-result-chunks.png)

Let's try to make a couple more questions.

- Question 2:

![app-result-2](/img/qa-gpt-app-streamlit-result-2.png)

- Question 3:

![app-result-3](/img/qa-gpt-app-streamlit-result-3.png)

- Question 4:

![app-result-4](/img/qa-gpt-app-streamlit-result-4.png)


