# SEI Cortex Demo

This example repo is based on a quickstart from Snowflake. The quickstart was archived, but the original version is located here: [QuickStart](https://github.com/Snowflake-Labs/sfquickstarts/blob/b792a6ae6d7f832782a92cb28e843ad57101e54b/site/sfguides/src/ask-questions-to-your-documents-using-rag-with-snowflake-cortex-search/ask_questions_to_your_own_documents_with_snowflake_cortex_search.md)

# Snowflake Cortex
<img width="1920" height="747" alt="image" src="https://github.com/user-attachments/assets/34e793bc-3f85-4cf9-94ca-12bf57299c02" />

[Snowflake Cortex](https://www.snowflake.com/en/product/features/cortex/) 


# RAG Overview
In a typical RAG application, there ia a process to chunk and embeed the content into a Vector store that is used later to retrieve the similar chunks of content, who are passed into a LLM to provide an answer:

<img width="7263" height="3196" alt="image" src="https://github.com/user-attachments/assets/076082f4-85ca-4413-b2fd-c7a4b5c79748" />

# Chunking Data for LLMs
### Why It Matters
* Improves retrieval relevance, understanding, and response quality
* Prevents exceeding token limits of models

Each LLM has a maximum context length (in tokens). For example:

* GPT-4: ~8k to 128k tokens (depending on version)
* GPT-3.5: ~4k tokens
* Claude 3 Opus: ~200k tokens

A token ≠ word. One word ≈ 1.3–1.5 tokens on average.

### Recommended Chunk Sizes by Use Case
| Use Case              | Chunk Size (Tokens) | Overlap (Tokens) | Notes                              |
|-----------------------|---------------------|------------------|------------------------------------|
| RAG / Semantic Search | 300–500             | 50–100           | Balances precision + context       |
| Document Summarization | 800–1200            | 100              | Longer = better flow               |
| Q&A Over Text         | 200–600             | 50–100           | Enables pinpointing answers        |
| Code Analysis         | 50–200 lines        | 0–20 lines       | Split by functions/logical blocks  |
| HTML / Web Content    | 300–1000            | 50               | Use semantic sections              |


### Tips
* Use semantic/structural splitting (not fixed length)
* Add overlap to avoid cutting off meaning
* Use tokenizers (e.g., tiktoken) to stay within limits

# Cortex
Snowflake Cortex Search is a fully managed indexing and retrieval service that simplifies and empower RAG applications. The service automatically creates embeddings and indexes and provides an enterprise search service that can be acessed via APIs:

<img width="973" height="369" alt="image" src="https://github.com/user-attachments/assets/5da1fdbe-80c1-4e0a-aec1-9e178c85eafd" />

# Get Started

## Prerequisites
1. Go to [Snowflake](https://www.snowflake.com)
2. Sign up for a personal temporary account. You do not need to add a credit card number. The account will last 30 days and you will have $400 of free credits to use. You can create a new free account after the 30 days is up. Create an AWS account in us-east-2 if able to select. 
3. Make sure you can log in to the account.
4. Download all documents from these 2 locations: [SEI Medical](https://sysev.sharepoint.com/:f:/r/sites/AllSEI/Shared%20Documents/Hello%20Services/HR%20Resources/Medical%20-%20Anthem%20Health%20Insurance?csf=1&web=1&e=XJgllw) and [SEI Dental](https://sysev.sharepoint.com/sites/AllSEI/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2FAllSEI%2FShared%20Documents%2FHello%20Services%2FHR%20Resources%2FDelta%20Dental%20Insurance&viewid=d12be2dd%2Db55f%2D4f33%2D9c84%2D54f4e1d3c556)

I did not download the sub folders under medical, but if you want to, you could try with these and see if we get different results. 

# Lab Instructions

## Upload your data to Snowflake

Open a new Snowsight SQL editor and run the following code: 

```sql
CREATE DATABASE SEI_MEDICAL_DENTAL;
CREATE SCHEMA DATA;

create or replace stage docs ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

ls @docs;
```

The ls @docs; command will not show any results. Navigate to your new database and schema, and click on the stage "docs". In the upper right corner of the dialog box, click to open the stage in a new window: 

<img width="370" height="112" alt="image" src="https://github.com/user-attachments/assets/b573ba4a-5576-437c-9105-a478e28824f7" />

Click add files, and drag or select files to upload. 

<img width="170" height="66" alt="image" src="https://github.com/user-attachments/assets/bad2220e-6562-4048-b461-0bce672fdb69" />

Specify the medical files to a medical directory and dental files to a dental directory. 

<img width="485" height="105" alt="image" src="https://github.com/user-attachments/assets/08c4a45f-f5f0-4879-999a-adab5f3c19b1" />

Close out this window and try ```ls @docs;``` again. You should see: 

<img width="765" height="274" alt="image" src="https://github.com/user-attachments/assets/dbbc8aa8-9b27-49df-9230-f0557be63af5" />

## Prepare your data for cortex search

1. Enable Cortex for our cloud provider and region

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US';
```

2. Create a temporary table with data and metadata from each document stored in the table from the stage
   
```sql
CREATE or replace TEMPORARY table RAW_TEXT AS
SELECT 
    RELATIVE_PATH,
    SIZE,
    FILE_URL,
    build_scoped_file_url(@docs, relative_path) as scoped_file_url,
    TO_VARCHAR (
        SNOWFLAKE.CORTEX.PARSE_DOCUMENT (
            '@docs',
            RELATIVE_PATH,
            {'mode': 'LAYOUT'} ):content
        ) AS EXTRACTED_LAYOUT 
FROM 
    DIRECTORY('@docs');
```

3. For each file, chunk the data into smaller pieces of text for search. Note: Try adjusting the chunk size to see if the result is better. See above recommendations. 

```sql
create or replace TABLE DOCS_CHUNKS_TABLE ( 
    RELATIVE_PATH VARCHAR(16777216), -- Relative path to the PDF file
    SIZE NUMBER(38,0), -- Size of the PDF
    FILE_URL VARCHAR(16777216), -- URL for the PDF
    SCOPED_FILE_URL VARCHAR(16777216), -- Scoped url (you can choose which one to keep depending on your use case)
    CHUNK VARCHAR(16777216), -- Piece of text
    CHUNK_INDEX INTEGER, -- Index for the text
    CATEGORY VARCHAR(16777216) -- Will hold the document category to enable filtering
);

insert into docs_chunks_table (relative_path, size, file_url,
                            scoped_file_url, chunk, chunk_index)

    select relative_path, 
            size,
            file_url, 
            scoped_file_url,
            c.value::TEXT as chunk,
            c.INDEX::INTEGER as chunk_index
            
    from 
        raw_text,
        LATERAL FLATTEN( input => SNOWFLAKE.CORTEX.SPLIT_TEXT_RECURSIVE_CHARACTER (
              EXTRACTED_LAYOUT,
              'markdown',
              1512,
              256,
              ['\n\n', '\n', ' ', '']
           )) c;
```

4. Add a category for each chunk of text (medical or dental). Note - if you didn't have the category as explicitely as we have the files organized, the cortex "classify_text" can apply a category based on the context in each chunk. 

```sql
 CREATE OR REPLACE TEMPORARY TABLE docs_categories AS WITH unique_documents AS (
  SELECT
    DISTINCT relative_path, chunk
  FROM
    docs_chunks_table
  WHERE 
    chunk_index = 0
  ),
 docs_category_cte AS (
  SELECT
    relative_path,
    TRIM(snowflake.cortex.CLASSIFY_TEXT (
      'Title:' || relative_path || 'Content:' || chunk, ['Dental', 'Medical']
     )['label'], '"') AS category
  FROM
    unique_documents
)
SELECT
  *
FROM
  docs_category_cte;
```

5. Update the chunk table with the categories found above. This will help the LLM find the relevant information quickly.

```sql
update docs_chunks_table 
  SET category = docs_categories.category
  from docs_categories
  where  docs_chunks_table.relative_path = docs_categories.relative_path;
```

6. Create the cortex search service that the LLM will use to query and find relevant information for each question

```sql
create or replace CORTEX SEARCH SERVICE CC_SEARCH_SERVICE_CS
ON chunk
ATTRIBUTES category
warehouse = COMPUTE_WH
TARGET_LAG = '1 minute'
as (
    select chunk,
        chunk_index,
        relative_path,
        file_url,
        category
    from docs_chunks_table
);
```

# Create Streamlit to Query Cortex Search

## What is Streamlit?
Streamlit is an open-source Python framework for building interactive web apps quickly and easily, especially for data science and machine learning use cases. It lets you turn Python scripts into shareable apps with widgets like sliders, dropdowns, and charts—all with minimal front-end code. Streamlit is popular for creating dashboards, prototypes, and tools that visualize and interact with data in real time.

Snowflake acquired Streamlit and now provides it as a native, hosted dashboarding solution under their "data-app" development framework Snowpark. 

## Create a streamlit app

In Snowflake, navigate to create a new Streamlit App: 

<img width="472" height="284" alt="image" src="https://github.com/user-attachments/assets/b1bcc9a2-4a98-4991-a75c-b856c69d9133" />

Choose a name for your app, and select the database and schema you used in the previous steps: 
<img width="589" height="415" alt="image" src="https://github.com/user-attachments/assets/55f329ca-852c-420d-9006-b3dddb0a8ee8" />

Copy and paste the streamlit.py code and environment.yml files into the streamlit app. 

Click packages and search for "snowflake.core". Click on snowflake.core to enable for your app. 

<img width="383" height="238" alt="image" src="https://github.com/user-attachments/assets/9dd4d39c-3986-44ea-8350-f26e6c73d17a" />

Click "Run". 

Navigate back to Snowflake Home and click on your streamlit app. 

<img width="1120" height="477" alt="image" src="https://github.com/user-attachments/assets/5003f4ba-b39e-42f6-bbc0-d3ad44edac25" />

<img width="1354" height="687" alt="image" src="https://github.com/user-attachments/assets/76b44df7-98b0-4a60-bd0f-d38c66b371e2" />

## Code Explanation
The streamlit code contains the UI elements to provide the drop down of categories, models, and whether to use SEI docs in the response. It also includes the prompt window to allow a user to ask a question. 

If the user has selected to use the RAG we developed in the previous steps, the logic will grab the relevant chunks to provide additional context to the model. It will add the retrieved context to a predefined prompt to give the model basic instructions. 

```python
    if st.session_state.rag == 1:
        prompt_context = get_similar_chunks_search_service(myquestion)
  
        prompt = f"""
           You are an expert chat assistance that extracs information from the CONTEXT provided
           between <context> and </context> tags.
           When ansering the question contained between <question> and </question> tags
           be concise and do not hallucinate. 
           If you don´t have the information just say so.
           Only anwer the question if you can extract it from the CONTEXT provideed.
           
           Do not mention the CONTEXT used in your answer.
    
           <context>          
           {prompt_context}
           </context>
           <question>  
           {myquestion}
           </question>
           Answer: 
           """

        json_data = json.loads(prompt_context)

        relative_paths = set(item['relative_path'] for item in json_data['results'])
```

The user question entered in the input box is added to the prompt above and sent to the command: 

```python
    prompt, relative_paths =create_prompt (myquestion)
    cmd = """
            select snowflake.cortex.complete(?, ?) as response
          """
    
    df_response = session.sql(cmd, params=[st.session_state.model_name, prompt]).collect()
```

to get a response with the chosen model. 

## Items to explore
1. Try searching without using the SEI docs
2. Try out different models
3. Observe the chunks that are returned and how relevant they are
4. Try creating smaller or larger chunks and see if the responses are better / worse.

# SEI Impacts
Rapidly evolving and innovating space - every vendor is releasing new products and features quickly! Models are improving rapidly, new vector databases, less detailed knowledge needed to operate. 

## Competitors
* AWS SageMaker (Bedrock, Textract, Jumpstart, etc)
* Azure Data Fabric
* GCP? Others?
* https://autotasks.io/

## Client Usage
* Agents
* Marketplace
* Structured / Unstructured data
* AI SQL editor
* Cortex Intelligence
* Openflow - Confluence, Sharepoint, etc?

## SEI Opportunities
* Data Strategy + Unstructured Data Mngt
* AI Governance and Security
* Others? 






 

