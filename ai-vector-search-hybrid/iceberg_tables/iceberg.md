# Similarity Search on Iceberg Tables

## Introduction

This lab walks you through the steps to run a similarity search on an Iceberg table and then create a vector index on the Iceberg table and repeat the similarity search.

Watch the video below for a quick walk-through of the Similarity Search on Iceberg Tables lab:

[Iceberg Similarity Search](https://videohub.oracle.com/media/Vector-Search-Exhaustive-Search-Lab/1_cmymq19w)

Estimated Time: X

### What is Apache Iceberg

Apache Iceberg is an open source table format designed to simplify the management of vast data lakes and data lakehouses while improving query performance.

Iceberg tables differ from traditional relational tables found in databases such as Postgres, MySQL, or Oracle. Relational tables store both metadata and data in the database where the data is processed and are well suited for structured application data. Moreover, strict relationships between tables in the database can be enforced. Iceberg tables, on the other hand, store both data and metadata in some form of file system storage layer, such as your local file system, Amazon S3, Google Cloud Storage, or Oracle Object Storage. This separation of data and metadata storage and compute decouples data processing from the data itself and gives end users the flexibility to choose the processing engine that is right for their specific needs.

For more information about Apache Iceberg tables see [What Is Apache Iceberg? Understanding Iceberg Tables](https://www.oracle.com/autonomous-database/apache-iceberg/).

### Part 1 - Similarity Search on Iceberg Tables

The first part of this Lab will focus on performing a similarity search on an Iceberg table. We will create an external table to access the Iceberg table data since it is located in Object Storage which is external to Oracle AI Database. The external table format supports the VECTOR data type and makes the access of Iceberg tables transparent within Oracle AI Database. Currently the DBMS_CLOUD.CREATE_EXTERNAL_TABLE API does not support the VECTOR datatype for Iceberg tables.

In this Lab we have already created the Iceberg table on OCI Object Storage. The table is based on Wikipedia data and the dataset is available on [Hugging Face](https://huggingface.co/datasets/CohereLabs/wikipedia-2023-11-embed-multilingual-v3). The dataset was created by CohereLabs using the [Cohere Embed V3 embedding model](https://txt.cohere.com/introducing-embed-v3/) to create the vector embeddings. We are just using a small 1000 article subset of the data for this Lab. The files on Hugging Face were distributed as Parquet files and we used Python and Spark SQL scripts to create the actual Iceberg table.

### Part 2 - Vector Indexes on Iceberg Tables (Vectors on Ice)

In the second part of this Lab you will create a Vector Index, also known as Vectors on Ice, on the external Iceberg table to significantly improve the search performance of the similarity search query we ran in the first part of the Lab. We will compare the SQL execution plan to the original plan that was generated in the first part of the Lab to show how you can verify that the vector index was used.

### Objectives

In this lab, you will:

* Run queries to discover the characteristics of an Iceberg table stored in OCI Object Store
* Run similarity search on an Iceberg table and investigate the database execution plan
* Create a vector index on the Iceberg table's VECTOR column
* Run another similarity search on the Iceberg table using the vector index and investigate the database execution plan to see how the index was used.

### Prerequisites

This lab assumes you have:

* An Oracle Account (oracle.com account)
* All previous labs successfully completed

## Task 1: Investigate the Iceberg Table

Iceberg tables can have two different formats, they can be Manifest-file based or they can be Catalog-backed. If you're interested in these two different formats more information is available here: [Apache Iceberg Tables Overview](https://docs.oracle.com/en/database/oracle/oracle-database/26/sutil/oracle_bigdata-accessing-apache-iceberg.html#GUID-C88E404B-77C1-45EF-BA2C-5F3F8CA1B3E3). In this Lab we will use a Manifest-file based Iceberg table.

1. The Iceberg table is stored in OCI Object Storage in a storage bucket. You can query different parts of the Iceberg table with the following SQL:

    ```[]
    <copy>
    SELECT *
    FROM DBMS_CLOUD.LIST_OBJECTS(
      'OCI_CRED2',
      'https://objectstorage.us-ashburn-1.oraclecloud.com/n/oradbclouducm/b/bucket-vector/o/iceberg/db/wiki_iceberg_1K'
    );
    </copy>
    ```

    The output should show you something similar to the following. Note the data/\*.parquet is the parquet file, or the actual data and the metadata/ files are the metadata that describes how to access the data.

    ```text
    OBJECT_NAME                                                                           BYTES
    ________________________________________________________________________________ __________
    data/00000-1-c585a711-e134-41a4-9b90-76ba51f092a1-0-00001.parquet                   2092091
    metadata/22ac8282-b036-4fb7-ba17-ef6d03ba055f-m0.avro                                  7398
    metadata/snap-1764402987059885972-1-22ac8282-b036-4fb7-ba17-ef6d03ba055f.avro          4468
    metadata/v1.metadata.json                                                              1899
    metadata/version-hint.text                                                                1
    ```

## Task 2: Create an external table

Next we will create an external table so that we can access our Iceberg table.

1. Run the following SQL to create an external table named EXT\_VECT\_TABLE:

    ```[]
    <copy>
    CREATE TABLE ext_vect_table (
      id    VARCHAR2(32),
      url   VARCHAR2(300),
      title VARCHAR2(200),
      text  CLOB,
      emb   VECTOR(1024, FLOAT32)
      )
      ORGANIZATION EXTERNAL (
        TYPE ORACLE_BIGDATA
        DEFAULT DIRECTORY DATA_PUMP_DIR
        ACCESS PARAMETERS (
          com.oracle.bigdata.fileformat=PARQUET
          com.oracle.bigdata.credential.name=OCI_CRED2
          com.oracle.bigdata.access_protocol=iceberg
          com.oracle.bigdata.log.opt=normal
          com.oracle.bigdata.debug=true
          com.oracle.bigdata.log.qc=DATA_PUMP_DIR:ext_qc_%p
          com.oracle.bigdata.log.exec=DATA_PUMP_DIR:ext_exec_%p_%a
      )
      LOCATION ('iceberg:https://objectstorage.us-ashburn-1.oraclecloud.com/n/oradbclouducm/b/bucket-vector/o/iceberg/db/wiki_iceberg_1K/metadata/v1.metadata.json')
    )
    REJECT LIMIT unlimited;
    </copy>
    ```

    Notice that we have explicitly specified the columns for the table using Oracle data types including the VECTOR data type for the vector embedded column. Also note that we have referenced the storage location in our Object Storage bucket for the Iceberg table.

## Task 3: Run a describe on the new table

In this task we will run a describe on the new table to verify the columns created.

1. Run the following to describe the table columns:

    ```[]
    <copy>
    DESC ext_vect_table
    </copy>
    ```

    The output should look like the following:

    ```text
    Name     Null?    Type
    ________ ________ _____________________________
    ID                VARCHAR2(32)
    URL               VARCHAR2(300)
    TITLE             VARCHAR2(200)
    TEXT              CLOB
    EMB               VECTOR(1024,FLOAT32,DENSE)
    ```

    Notice the EMB column has a VECTOR datatype.

## Task 4: Run a similarity search on the Iceberg Table

In this task we will run a similarity search on the Iceberg table data by accessing the external table we just created.

1. Run the following statement to set the parameters to enable the creation of the query vector. In this Lab we are using the OCI Gen AI service to access the same "cohere.embed-multilingual-v3.0" embedding model that was used to create the data vectors in the Iceberg table:

    ```[]
    <copy>
    var params clob;
    begin
      :params := '{
        "provider": "ocigenai",
        "credential_name": "OCI_GENAI_CRED",
        "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
        "model": "cohere.embed-multilingual-v3.0",
        "transfer_timeout":1200
      }';
    end;
    /
    </copy>
    ```

2. Run the following query to find Wikipedia articles about football:

    ```[]
    <copy>
    SELECT
       url,
       title,
       vector_distance(emb, dbms_vector_chain.utl_to_embedding('football', JSON(:params))) AS dist,
       text
    FROM  ext_vect_table
    ORDER BY dist DESC
    FETCH FIRST 5 ROWS ONLY;
    </copy>
    ```

    ![exhaustive query2](images/parks_exhaustive_rock_climbing.png " ")

    We have included the vector distance, that is the distance between the 'football' vector and the data vector in descending order. The closest match is first and then the next four matches in descending order.

3. The last step in this task is to take a look at the execution plan. Since we have not created any indexes we will expect to see a FULL TABLE SCAN of the EXT\_VECT\_TABLE.

    ```[]
    <copy>
    SELECT *
    FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR());
    </copy>
    ```

    ![distance query](images/parks_exhaustive_rock_climbing_distance.png " ")

## Task 5: Create a vector index on the Iceberg Table

In this task we will run a similarity search on the Iceberg table data by accessing the external table we just created.

1. Run the following statement to create a vector index on the EMB column in the EXT_VECT_TABLE, which is the VECTOR column where the vector embeddings for the TEXT column are stored. Note that although the Iceberg table is stored in Object Storage the vector index will be stored in Oracle AI Database.

    ```[]
    <copy>
    ALTER SESSION SET "_xt_table_hidden_column" = TRUE;
    ALTER SESSION SET "_xt_hybrid_addl_hidden_column" = TRUE;
    CREATE VECTOR INDEX ext_vect_idx_ivf ON ext_vect_table (emb)
    ORGANIZATION NEIGHBOR PARTITIONS;
    </copy>
    ```

  Note that we added the session level parameters again just in case you might have exited your previous session where we set them before we created the external table EXT\_VECT\_TABLE.

## Task 6: Run the similarity search a second time

Now that you have created a vector index you can run the same similarity search you ran in Task 4. Now the query execution should take advantage of the vector index and run much faster and access many fewer vectors.

1. Run the following statement to set the parameters to enable the creation of the query vector. This is the same statement that we ran in Task 4, Step 1:

    ```[]
    <copy>
    var params clob;
    begin
      :params := '{
        "provider": "ocigenai",
        "credential_name": "OCI_GENAI_CRED",
        "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
        "model": "cohere.embed-multilingual-v3.0",
        "transfer_timeout":1200
      }';
    end;
    /
    </copy>
    ```

2. Run the following query to find Wikipedia articles about football:

    ```[]
    <copy>
    SELECT
       url,
       title,
       vector_distance(emb, dbms_vector_chain.utl_to_embedding('football', JSON(:params))) AS dist,
       text
    FROM  ext_vect_table
    ORDER BY dist DESC
    FETCH FIRST 5 ROWS ONLY;
    </copy>
    ```

    ![exhaustive query2](images/parks_exhaustive_rock_climbing.png " ")

    This time the query should have run much faster.

3. The last step in this task is to take a look at the execution plan. Since we have not created any indexes we will expect to see a FULL TABLE SCAN of the EXT\_VECT\_TABLE.

    ```[]
    <copy>
    SELECT *
    FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR());
    </copy>
    ```

    ![distance query](images/parks_exhaustive_rock_climbing_distance.png " ")

## Learn More

* [Oracle AI Vector Search Users Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/index.html)
* [OML4Py: Leveraging ONNX and Hugging Face for AI Vector Search](https://blogs.oracle.com/machinelearning/post/oml4py-leveraging-onnx-and-hugging-face-for-advanced-ai-vector-search)
* [Oracle Database 26ai Release Notes](https://docs.oracle.com/en/database/oracle/oracle-database/23/rnrdm/index.html)
* [Oracle Documentation](http://docs.oracle.com)

## Acknowledgements

* **Author** - Andy Rivenes, Product Manager, AI Vector Search
* **Contributors**
* **Last Updated By/Date** - Andy Rivenes, Product Manager, AI Vector Search, February 2026
