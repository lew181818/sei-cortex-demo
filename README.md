# SEI Cortex Demo

This example repo is based on a quickstart from Snowflake. The quickstart was archived, but the original version is located here: [QuickStart](https://github.com/Snowflake-Labs/sfquickstarts/blob/b792a6ae6d7f832782a92cb28e843ad57101e54b/site/sfguides/src/ask-questions-to-your-documents-using-rag-with-snowflake-cortex-search/ask_questions_to_your_own_documents_with_snowflake_cortex_search.md)

# RAG Overview
In a typical RAG application, there ia a process to chunk and embeed the content into a Vector store that is used later to retrieve the similar chunks of content, who are passed into a LLM to provide an answer:

<img width="7263" height="3196" alt="image" src="https://github.com/user-attachments/assets/076082f4-85ca-4413-b2fd-c7a4b5c79748" />


# Cortex
Snowflake Cortex Search is a fully managed indexing and retrieval service that simplifies and empower RAG applications. The service automatically creates embeddings and indexes and provides an enterprise search service that can be acessed via APIs:

<img width="973" height="369" alt="image" src="https://github.com/user-attachments/assets/5da1fdbe-80c1-4e0a-aec1-9e178c85eafd" />

# Get Started

## Prerequisites
1. Go to [Snowflake](www.snowflake.com)
2. Sign up for a personal temporary account. You do not need to add a credit card number. The account will last 30 days and you will have $400 of free credits to use. You can create a new free account after the 30 days is up.
3. Make sure you can log in to the account.
4. Download all documents from these 2 locations: [SEI Medical](https://sysev.sharepoint.com/:f:/r/sites/AllSEI/Shared%20Documents/Hello%20Services/HR%20Resources/Medical%20-%20Anthem%20Health%20Insurance?csf=1&web=1&e=XJgllw) and [SEI Dental](https://sysev.sharepoint.com/sites/AllSEI/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2FAllSEI%2FShared%20Documents%2FHello%20Services%2FHR%20Resources%2FDelta%20Dental%20Insurance&viewid=d12be2dd%2Db55f%2D4f33%2D9c84%2D54f4e1d3c556)

I did not download the sub folders under medical, but if you want to, you could try with these and see if we get different results. 

# Lab Instructions

## Upload your data to Snowflake

Open a new Snowsight SQL editor and run the following code: 

```
CREATE DATABASE SEI_MEDICAL_DENTAL;
CREATE SCHEMA DATA;

create or replace stage docs ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

ls @docs;
```

The ls @docs; command will not show any results. Navigate to your new database and schema, and click on the stage "docs". In the upper right corner of the dialog box, click to open the stage in a new window: <img width="370" height="112" alt="image" src="https://github.com/user-attachments/assets/b573ba4a-5576-437c-9105-a478e28824f7" />
 

