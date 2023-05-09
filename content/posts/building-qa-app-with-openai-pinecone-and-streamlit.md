---
title: "Building a Q&A app capable of answering questions related to your enterprise documents using Azure OpenAI's GPT-4, Pinecone and Streamlit."
date: 2023-05-08T16:53:31+02:00
tags: ["python", "openai", "ai", "azure", "embeddings", "llm"]
description: "The purpose of this post is to show you how to build a basic GPT-4 Q&A app in just a couple of hours that is capable of answering questions about your company's internal documents. We will use Azure OpenAI, Pinecone and Streamlit to build it."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/building-qa-app-with-openai-pinecone-and-streamlit).


I won't tell you anything new if I say that there's currently a boom of AI-related topics. Companies are frantically searching for use cases on how to integrate GPT or other LLM models into their ecosystem. Right now, everyone seems to be interested in working with LLM models in one way or another.

One of the questions that I have been receiving quite frequently lately is how to build an application that uses a language model such as GPT-4 and is capable of answering questions about internal company topics.

GPT models have extensive knowledge, but it's obviously a generalized knowledge. If you want to ask very specific questions about internal workflows, it's impossible for them to know the answers as they haven't been trained for them.

Setting up an application that uses GPT-4 (the latest version of GPT) and is able to answer internal questions about your company's operations may seem like a complicated task, but the truth is that it's stupidly simple once you understand the three or four most important concepts.    

And that's precisely what I want to show you in this post. It may not be the most complex application out there, but in less than 2 hours, we will build an app that uses GPT-4 and semantic search to answer internal questions about your company.

But before we start coding, I think it's necessary to explain a couple of concepts to better understand what we're going to do here.


# **How can GPT-4 respond to a question about topics it doesn't know about?**

We have two options for enabling our LLM model to understand and answer our private domain-specific questions:

- **Fine-tune** the LLM on text data covering the topic mentioned.
- Using **Retrieval Augmented Generation (RAG)**, a technique that implements an information retrieval component to the generation process. Allowing us to retrieve relevant information and feed this information into the generation model as a secondary source of information.

We will go with option 2.

The RAG process might sound daunting at first, but it really is not. The process is quite simple once we try to explain it in a simple way, here’s a summary of how it works:
- Run a semantic search against your knowledge database to find content that looks like it could be relevant to the user’s question.
- Construct a prompt consisting of the data extracted from the knowledge database, followed by "Given the above content, answer the following question" and then the user’s question.
- Send the prompt through to GPT-4 and see what answer comes back.

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

# **Knowledge database and embeddings**

With RAG the retrieval of relevant information requires an external "Knowledge database", a place where we can store and use to efficiently retrieve information.    
We can think of this database as the external long-term memory of our LLM.

To retrieve information that is semantically related to our questions, we're going to use "vector embeddings". 

## **What are vector embeddings?**

Vector embeddings are a way to represent words and whole sentences in a numerical manner. 

We understand that computers comprehend numerical language, and as a result, we attempt to encode words in a sentence into numbers to enable the computer to read and process them. 

However, we require computers to accomplish more than just reading and processing. We need them to establish connections between each word in a sentence or document, capture its context, and recognize its semantic and syntactic features and similarities.

To achieve this level of semantic and syntactic relationship, we must go beyond simply mapping a word to a set of numbers. We require a more extensive representation of these numbers that can reflect both the semantic and syntactic properties of the text.

We need vectors embeddings.

To create these vectors embeddings and store it in a knowledge database, we follow a simple three-step process:
1. Retrieve a paragraph from the document that we want to store in the knowledge database.
2. Use the ``text-embedding-ada-002`` model to convert it into a vector embeddings.
3. Store the resulting vector embeddings into the knowledge database.

We then repeat these three steps for each document in the set until all of them have been stored in the knowledge database.


## **Knowledge database**

Vector databases are used to store and query vectors efficiently. They allow us to search for similar vectors based on their similarity in a high-dimensional space.

We're going to use one of those vector database as our knowledge database, right now there are quite a few options available, but in this post, we will use [Pinecone](https://www.pinecone.io/) as our vector database.

Why choose Pinecone? Because it is a serverless vector database, which means that there is no infrastructure and maintainance required.


# **How the app works**

After breaking down what I believe are the most important concepts on the topic, let's start developing our app. 

To build it, we're going to make use of just 3 technologies:
- Azure OpenAI to interact with the LLM models.
- Pinecone to set up a knowledge database.
- Streamlit to build a simple user interface.

The following diagram shows what we are going to build.

![app-diagram](/img/qa-gpt-app-diagram.png)

To simplify the development of the application, we will divide the app functionality into 2 processes:
- An **ingestion process** that will be responsible for:
    - Reading our private documents (PDFs, words, wiki, JIRA, or whatever kind of document we want to ingest).
    - Converting the docs into vector embeddings using Azure OpenAI ``text-embedding-ada-002`` model.
    - Saving the vector embeddings into Pinecone.
- An **application that allows a user to ask questions about the ingested data**. This application has to:
    - Transform the user's query into a vector embedding using Azure OpenAI ``text-embedding-ada-002`` model.
    - Retrieve the relevant information to the given query from PineCone.
    - Assemble a prompt.
    - Send the prompt through to GPT-4 and get the answer that comes back.


# **Building the ingest process**

Before start writing code, we have to create a Pinecone database, a Pinecone Index and deploy a ``text-embedding-ada-002`` model in our ``Azure OpenAI`` instance.

## **1. Create the Pinecone index**

Let's start by creating the knowledge database.

Pinecone offers a free plan that let's you use the fully managed service for small workloads without any cost, so that's a good option if you just want to tinker with it for a while and nothing more.

Once you have sign-in, you have to create a new index using cosine as metric and 1536 dimensions. 

- Why 1536 dimensions specifically? Because the embedding model we will use to create the vectors, ``text-embedding-ada-002``, outputs vectors with 1536 dimensions. 

- The cosine metric will be used to calculate similarity between vectors.

![pinecone-creation](/img/qa-gpt-app-pinecone-index-creation.png)

After waiting a little bit the Pinecone Index we'll be ready for use.

![pinecone-ready](/img/qa-gpt-app-pinecone-index.png)


## **2. Deploy a text-embedding-ada-002 model on Azure OpenAI**

The next step will be to deploy a ``text-embedding-ada-002`` model on ``Azure OpenAI``. This model will be used to transform text into vector embeddings.

Go to Azure OpenAI > Navigate to "Model Deployments" and deploy a ``text-embedding-ada-002`` model.

![model-deployment](/img/qa-gpt-app-openai-deployments.png)


## **3. Build the ingest data process using LangChain**

> If you want to take a closer look at the source code or want to try running it on your machine, you can visit my [Github](https://github.com/karlospn/building-qa-app-with-openai-pinecone-and-streamlit/blob/main/Ingest%20data%20into%20Pinecone.ipynb) where you'll find a ``Jupyter Notebook`` that runs the ingest process.

These are the steps involved in the ingest process:
- Read our private documents (PDFs, words, wiki, JIRA, or whatever kind of document we want to ingest).
- Break the documents down into smaller text chunks.
- Convert the chunks of text into vector embeddings using the Azure OpenAI ``text-embedding-ada-002`` model.
- Store the resulting vector embeddings in Pinecone.

Let's break down step by step the data ingestion process into Pinecone.

### **3.1. Read the document files and split them into chunks**

The first step is to read whatever internal documents we want to ingest (for this example I'm using the Microsoft Microservices book) and split them into multiple chunks of text.

- To read and parse the documents, I will be using the [LangChain](https://python.langchain.com/en/latest/index.html) library.

We split the documents into multiple chunks so that the semantic search is able to return only the paragraphs of information that are relevant to our query.   
It would not make any sense to store entire documents without splitting them into multiple chunks, since any semantic search would then return the entire content of the book/document.

For this example and to make it as simple as possible, I am simply splitting the document into chunks of 1000 characters, but if we wanted to do it better we could, for example, split it by paragraphs.

```python
loader = UnstructuredPDFLoader("./docs/NET-Microservices-Architecture-for-Containerized-NET-Applications.pdf")
data = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=0)
chunks = text_splitter.split_documents(data)

print (f'You have a total of {len(chunks)} chunks')
```

### **3.2. Iniatilize OpenAI client and PineCone client**

The second step is to iniatilize the OpenAI client and the PineCone client

There are a few parameters needed to initalize the OpenAI client:

- ``AZURE_OPENAI_APIKEY``: Azure OpenAI ApiKey.
- ``AZURE_OPENAI_BASE_URI``: Azure OpenAI URI.
- ``AZURE_OPENAI_EMBEDDINGS_MODEL_NAME``: The ``text-embedding-ada-002`` model deployment name.

To use the ``openai`` python library alongside with Azure OpenAI, the parameter ``openai_api_type`` must be always set to ``azure`` and the parameter ``chunk_size`` must be always set to ``1``.

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

The last step is to convert all the chunks of text into vector embeddings and store them into Pinecone

This last part may seem complicated, but actually it's quite simple. With just one line of code and using the Pinecone client, we can do it.

```python
docsearch = Pinecone.from_texts([t.page_content for t in chunks], embeddings, index_name=PINECONE_INDEX_NAME)
```

As you can see, the entire process of converting a few internal documents into vectors and saving them in Pinecone is super simple, it's no more than 10 lines of code! 

Now that we have the data stored in our knowledge database, let's build an application that makes use of it.

# **Building the query app**

Before start writing the app, we have to deploy a ``gpt-4`` model in our ``Azure OpenAI`` instance.

## **1. Deploy a GPT-4 model on Azure OpenAI**

Let's start by deploying a ``gpt-4`` LLM model on ``Azure OpenAI``. 
This LLM model will be used to generate the response to the user questions.

Go to Azure OpenAI > Navigate to "Model Deployments" and deploy a ``gpt-4`` or ``gpt-4-32k`` model.

![model-deployment](/img/qa-gpt-app-openai-deployments.png)

## **2. Build the querying app using Streamlit**

We are going to set up a simple form with a text input field where the user can type the question they want and a submit button.    
Once the user types the question in the text input and presses the submit button, the following steps will be executed:
- Transform the user's query into a vector embedding using Azure OpenAI ``text-embedding-ada-002`` model.
- Retrieve the relevant information to the user query from PineCone.
- Assemble a prompt.
- Send the prompt through to GPT-4 and get the answer that comes back.

To build the UI we're using [Streamlit](https://streamlit.io/). I decided to use Streamlit because I can build a simple and functional form with just a few lines of Python.

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


# Add the UI title, text input and search button
st.title("Q&A App 🔎")
query = st.text_input("What do you want to know?")

if st.button("Search"):
   
    # Get Pinecone API environment variables
    pinecone_api = os.getenv('PINECONE_API_KEY')
    pinecone_env = os.getenv('PINECONE_ENVIRONMENT')
    pinecone_index = os.getenv('PINECONE_INDEX')
    
    # Get Azure OpenAI environment variables
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
 
    # Search for the 3 most similar vectors in Pinecone
    search_response = index.query(
        top_k=3,
        vector=query_vector,
        include_metadata=True)

    # Combine the 3 vectors into a single text variable that it will be added in the prompt
    chunks = [item["metadata"]['text'] for item in search_response['matches']]
    joined_chunks = "\n".join(chunks)

    # Write which are the selected chunks in the UI for debugging purposes
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

As you may have seen in the code block above, the entirety of the query app is no more than 100 lines of code and is quite simple to understand. 

But nonetheless, I'll try to explain the most relevant parts.

## **2.1. Transform the user's query into a vector embedding**

To transform the user question into a vector embedding, we'll be using the ``text-embedding-ada-002`` model from Azure OpenAI alongside with the ``openai`` Python package.

To create the embedding we just have to invoke the ``Embedding.create`` method specifying the model that will be used to generate the vector and the user query we want to convert.

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

The ``index.query`` method from the Pinecone client allow us to retrieve the relevant information from the knowledge database.    
The ``top_k`` parameter is used to specify how many vectors we want to retrieve, in this case we're returning the 3 vectors that are most similar to our question.

- Why did we retrieve 3 vectors from our knowledge base instead of just one? 

When we saved the document, we partitioned the data into multiple chunks, so it's possible that the complete answer to our question is not located in just one vector but in more than one. That's why in this case we retrieve 3 vectors.

Once we have retrieved the vectors from Pinecone, we group them into a single 'string'. This 'string' represents the context that we will add to the prompt and in which we will specify to GPT-4 that it can only respond using the information given in this context, and in no case can it make up any data.

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

The last step is sending the prompt to GPT-4 using the ``ChatCompletion.create`` method from the ``openai`` Python package and get the answer that comes back.

The GPT-4 model accepts messages formatted as a conversation. The messages parameter must be an array of dictionaries with represents a conversation organized by role.

The system role also known as the system message must be included at the beginning of the array. This message provides the initial instructions to the model. 

After the system role, you can include a series of messages between the user and the assistant, in this case we are adding the prompt that we have built in the previous section.

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

And finally, all we have to do is to obtain the response returned by GPT-4 and display it on the screen.

# **6. Testing the query app**

Let's test if the query app works correctly. 

> Remember that the document we used to fill the knowledge base is the Microsoft .NET Microservices book, so all questions we ask must have to be about that spefici topic.

- Question 1:

![app-result-1](/img/qa-gpt-app-streamlit-result-1.png)

For every question we can also inspect whichs chunks of text are being retrieved from Pinecone and feed to GPT-4 to build the response.

![app-result-chunk](/img/qa-gpt-app-streamlit-result-chunks.png)

Let's try to make a couple more questions.

- Question 2:

![app-result-2](/img/qa-gpt-app-streamlit-result-2.png)

- Question 3:

![app-result-3](/img/qa-gpt-app-streamlit-result-3.png)

- Question 4:

![app-result-4](/img/qa-gpt-app-streamlit-result-4.png)


