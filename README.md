# SEI Cortex Demo

This example repo is based on a quickstart from Snowflake. The quickstart was archived, but the original version is located here: [QuickStart](https://github.com/Snowflake-Labs/sfquickstarts/blob/b792a6ae6d7f832782a92cb28e843ad57101e54b/site/sfguides/src/ask-questions-to-your-documents-using-rag-with-snowflake-cortex-search/ask_questions_to_your_own_documents_with_snowflake_cortex_search.md)

# RAG Overview
In a typical RAG application, there ia a process to chunk and embeed the content into a Vector store that is used later to retrieve the similar chunks of content, who are passed into a LLM to provide an answer:

<img width="7263" height="3196" alt="image" src="https://github.com/user-attachments/assets/076082f4-85ca-4413-b2fd-c7a4b5c79748" />


# Cortex
Snowflake Cortex Search is a fully managed indexing and retrieval service that simplifies and empower RAG applications. The service automatically creates embeddings and indexes and provides an enterprise search service that can be acessed via APIs:

<img width="973" height="369" alt="image" src="https://github.com/user-attachments/assets/5da1fdbe-80c1-4e0a-aec1-9e178c85eafd" />
