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


# **How can GPT-4 respond to a question about topics it doesn't know about?**

We have two options for enabling our LLM to understand and answer our private domain-specific questions:

- **Fine-tune** the LLM on text data covering the topic mentioned.
- Using **Retrieval Augmented Generation (RAG)**, a technique that implements an information retrieval component to the generation process. Allowing us to retrieve relevant information and feed this information into the generation model as a secondary source of information.

We will go with option 2.

The implementation might seem daunting at first, but it really is not, here’s a summary of how the entire process will look like:
- Run a semantic search against your knowledge database to find content that looks like it could be relevant to the user’s question.
- Construct a prompt consisting of the data extracted from the knowledge database, followed by "Given the above content, answer the following question" and then the user’s question.
- Send the prompt through to GPT-4 and see what answer comes back.

And that's it, what you're doing is nothing more than extracting the relevant data from a database and building a prompt like this one

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

# **Knowledge database and embeddings**

With RAG the retrieval of relevant information requires an external "Knowledge database", a place where we can store and use to efficiently retrieve information. We can think of this as the external long-term memory of our LLM.

We will need to retrieve information that is semantically related to our queries, to do this we need to use "vector embeddings". 

## **What are embeddings?**

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


## **Knowledge database**

Vector databases are used to store and query vectors efficiently. They allow us to search for similar vectors based on their similarity in a high-dimensional space.

We're going to use one of those vector database as our knowledge database, right now there are quite a few options available, but in this post, we will use Pinecone as our vector database.

Why choose Pinecone? Because it is a serverless vector database, which means that there is no maintainance required.


# **How the app works**

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


# **Building the ingest process**

Before start writing code, let's create a Pinecone database and deploy a ``text-embedding-ada-002`` model in our ``Azure OpenAI`` instance.

## **1. Create the Pinecone index**

Let's start by creating the knowledge database.

Pinecone offers a free plan that let's you use the fully managed service for small workloads without any cost, so that's a good option if you just want to tinker with it for a while and nothing more.

Once you have sign-in, you have to create a new index using cosine as metric and 1536 dimensions. Like this:
![pinecone-creation](/img/qa-gpt-app-pinecone-index-creation.png)

We're going to need 1536 dimensions because the embedding model we will use to create the vectors, ``text-embedding-ada-002`, outputs vectors with 1536 dimensions. 

The cosine metric is used to calculate similarity between vectors.

After waiting a little bit the index we'll be marked as ready for use.

![pinecone-ready](/img/qa-gpt-app-pinecone-index.png)


## **2. Deploy a text-embedding-ada-002 model on Azure OpenAI**

The next step will be to deploy a ``text-embedding-ada-002`` LLM model on ``Azure OpenAI``. This LLM model will be use to transform the text into vector embeddings.

Go to Azure OpenAI > Navigate to "Model Deployments" and deploy a ``text-embedding-ada-002`` model.

![model-deployment](/img/qa-gpt-app-openai-deployments.png)


## **3. Build the ingest data process using LangChain**

> If you want to take a closer look at the source code or want to try running it on your machine, you can visit my [Github](https://github.com/karlospn/building-qa-app-with-openai-pinecone-and-streamlit/blob/main/Ingest%20data%20into%20Pinecone.ipynb) where I have uploaded a ``Jupyter Notebook``.


These are the required steps that this process must execute.
- Read our private documents (PDFs, words, wiki, JIRA, or whatever kind of document we want to ingest).
- Convert the docs into vector embeddings using ``Azure OpenAI text-embedding-ada-002`` LLM model.
- Save the vector embeddings into Pinecone.

Let's break down step by step the process we are going to follow to ingest the data into Pinecone.

### **3.1. Read the document files and split them into chunks**

The first step is to read the doc files (for this example I'm using the .NET Microservices book) and split it into multiple chunks

To read and parse the documents, I will use the [LangChain](https://python.langchain.com/en/latest/index.html) library.

We split the documents into multiple chunks so that the semantic search is able to return only the paragraphs of information that belong to our query. It would not make any sense to store the documents without splitting them into multiple chunks, since any semantic search would then return the entire content of the book/document.

For this example and to make it as simple as possible, I am simply splitting the document into chunks of 1000 characters, but if we wanted to do it better we could, for example, split it by paragraphs.

```python
loader = UnstructuredPDFLoader("./docs/NET-Microservices-Architecture-for-Containerized-NET-Applications.pdf")
data = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=0)
chunks = text_splitter.split_documents(data)

print (f'You have a total of {len(chunks)} chunks')
```

### **3.2. Iniatilize Azure OpenAI embeddings client and PineCone client**

The second step is to iniatilize the Azure OpenAI embeddings client and the PineCone client

There are a few parameters needed to initalize the OpenAI Embeddings client:

- ``AZURE_OPENAI_APIKEY``: Azure OpenAI ApiKey.
- ``AZURE_OPENAI_BASE_URI``: Azure OpenAI URI.
- ``AZURE_OPENAI_EMBEDDINGS_MODEL_NAME``: The ``text-embedding-ada-002`` model deployment name.

To use the ``openai`` python library the parameter ``openai_api_type`` must be set to ``azure`` and the parameter ``chunk_size`` must be set to ``1``.

There is also a couple parameters required to initialize the Pinecone client:

- ``PINECONE_API_KEY``: Pinecone ApiKey.
- ``PINECONE_ENVIRONMENT``: Pinecone index environment.

```python 
embeddings = OpenAIEmbeddings(
    openai_api_base=AZURE_OPENAI_API_BASE, 
    openai_api_key=AZURE_OPENAI_API_KEY, 
    model=AZURE_OPENAI_EMBEDDINGS_MODEL_NAME, 
    openai_api_type='azure',
    chunk_size=1)

pinecone.init(
    api_key=PINECONE_API_KEY,
    environment=PINECONE_API_ENV  
)
```

### **3.3. Convert the chunks of text into vector embeddings and store it into Pinecone**

The last step is to convert the chunks of text into vector embeddings and store it into Pinecone

This last part may seem complicated, but actually it's quite simple. With just one line of code and using the Pinecone client, we can do it.

```python
docsearch = Pinecone.from_texts([t.page_content for t in chunks], embeddings, index_name=PINECONE_INDEX_NAME)
```

As you can see, the process of converting documents into vectors and saving them in Pinecone is super simple, just about 10 lines of code. 

Now that we have the data stored in our knowledge database, let's build the query application.

# **Building the query app**

Before start writing the app, let's deploy a ``gpt-4`` model in our ``Azure OpenAI`` instance.

## **1. Deploy a GPT-4 model on Azure OpenAI**

The start by deploying a ``gpt-4`` LLM model on ``Azure OpenAI``. 
This LLM model will be use to generate the query response.

Go to Azure OpenAI > Navigate to "Model Deployments" and deploy a ``gpt-4`` or ``gpt-4-32k`` model.

![model-deployment](/img/qa-gpt-app-openai-deployments.png)

## **2. Build the querying app using Streamlit**

Once the user types the question in the text input and presses the submit button, the following steps will be executed:
- Transform the user's query into a vector embedding using ``Azure OpenAI text-embedding-ada-002`` LLM model.
- Retrieve the relevant information from PineCone.
- Assemble the prompt.
- Send the prompt through to GPT-4 and get the answer that comes back.

To build the application user interfact we're using [Streamlit](https://streamlit.io/). I decided to use Streamlit because we I have a simple and functional UI with just a few lines of Python.

Let me show you the final result.

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


# Add the UI pieces
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
 
            # Get the response from GPT-4
            st.markdown("### Answer:")
            st.write(response.choices[0]['message']['content'])
   
   
        except Exception as e:
            st.error(f"Error with OpenAI Chat Completion: {e}")
```

As you may have seen in the code block above, the actions we perform are quite simple to understand. But I'll try to explain the most relevant ones.

## **2.1. Transform the user's query into a vector embedding**

To transform the query into a vector embedding, we'll be using the ``text-embedding-ada-002`` LLM model from Azure OpenAI alongside with the ``openai`` Python package.

We just have to invoke the ``Embedding.create`` method specifying the LLM model used to generate the vector and the user query we want to convert.

```python
# Convert your query into a vector using Azure OpenAI
try:
    query_vector = openai.Embedding.create(
        input=query,
        engine=embeddings_model_name,
    )["data"][0]["embedding"]
except Exception as e:
    st.error(f"Error calling OpenAI Embedding API: {e}")
    st.stop()
```

## **2.2. Retrieve the relevant information from Pinecone and assemble the prompt**

The ``index`` method from the Pinecone client allow us to retrieve the relevant information from our query knowledge database. The ``top_k`` parameter is used to specify how many vector we want to retrieve, in this case we're returning the 3 vector that are most similar to our question.

Why did we retrieve 3 vectors from our knowledge base instead of just 1? When we saved the document, we partitioned the data into multiple chunk, so it's possible that the complete answer to our question is not located in just one vector but in more than one. That's why in this case we retrieve 3 vectors.

Once we have retrieved the vectors from Pinecone, we group them into a single 'string'. This 'string' contains the context that we will add to the prompt and in which we will specify to GPT4 that it can only respond using the information given in this context, and in no case can it make up any data.

```python
search_response = index.query(
    top_k=3,
    vector=query_vector,
    include_metadata=True)

chunks = [item["metadata"]['text'] for item in search_response['matches']]

# Combine texts into a single chunk to insert in the prompt
joined_chunks = "\n".join(chunks)

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
```

## **2.2. Send the prompt through to GPT-4 and get the answer that comes back**

The last step is sending the prompt to GPT-4 and get the answer that comes back.

The GPT-4 model accepts messages formatted as a conversation. The messages parameter takes an array of dictionaries with represents a conversation organized by role.

The system role also known as the system message is included at the beginning of the array. This message provides the initial instructions to the model. 

After the system role, you can include a series of messages between the user and the assistant.

```python
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

# Get the response from GPT-4
st.markdown("### Answer:")
st.write(response.choices[0]['message']['content'])
```

And that's it the app is done.

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


