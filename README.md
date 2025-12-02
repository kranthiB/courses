# Course Catalog

A comprehensive collection of data engineering courses covering PySpark, Databricks, Data Warehousing, Spark Streaming, and hands-on projects.

---

## [PySpark](PySpark/)
A complete course on Apache Spark with Python, covering fundamentals to advanced optimization techniques.

| Module | Topic | Description |
|--------|-------|-------------|
| 01 | [Introduction to PySpark](PySpark/01_Introduction_to_PySpark.ipynb) | Overview of Apache Spark, its ecosystem, and why PySpark is essential for big data processing |
| 02 | [Spark Architecture and Execution Flow](PySpark/02_Spark_Architecture_and_Execution_Flow.ipynb) | Understanding Spark's driver-executor architecture and how jobs are executed |
| 03 | [Partitions, Transformations and Lazy Evaluation](PySpark/03_Partitions_Transformations_and_Lazy_Evaluation.ipynb) | Core concepts of data partitioning, narrow/wide transformations, and lazy evaluation strategy |
| 04 | [DataFrames and Execution Plans](PySpark/04_DataFrames_and_Execution_Plans.ipynb) | Working with DataFrames API and understanding physical/logical execution plans |
| 05 | [Environment Setup - Docker](PySpark/05_Environment_Setup_Docker.ipynb) | Setting up a local PySpark development environment using Docker containers |
| 06 | [First Spark Application - Hands On](PySpark/06_First_Spark_Application_Hands_On.ipynb) | Building and running your first Spark application with practical examples |
| 07 | [DataFrame Columns and Transformations](PySpark/07_DataFrame_Columns_and_Transformations.ipynb) | Working with DataFrame columns, selecting, renaming, and basic transformations |
| 08 | [Columns Transforms and Filters](PySpark/08_Columns_Transforms_and_Filters.ipynb) | Advanced column operations, filtering data, and conditional logic |
| 09 | [Strings, Dates and Nulls](PySpark/09_Strings_Dates_and_Nulls.ipynb) | Handling string operations, date/time manipulation, and null value management |
| 10 | [Union, Sorting and Aggregations](PySpark/10_Union_Sorting_and_Aggregations.ipynb) | Combining datasets, sorting operations, and performing aggregations (sum, avg, count) |
| 11 | [Window Functions and Unique Data](PySpark/11_Window_Functions_and_Unique_Data.ipynb) | Advanced analytics with window functions (rank, lead, lag) and removing duplicates |
| 12 | [Repartition, Coalesce and Joins](PySpark/12_Repartition_Coalesce_and_Joins.ipynb) | Managing data partitions and implementing various join operations (inner, outer, cross) |
| 13 | [Reading CSVs and Handling Bad Records](PySpark/13_Reading_CSVs_and_Handling_Bad_Records.ipynb) | Reading CSV files with different options and strategies for handling corrupt records |
| 14 | [Complex File Formats - Parquet, ORC](PySpark/14_Complex_File_Formats_Parquet_ORC.ipynb) | Working with columnar file formats like Parquet and ORC for optimized storage |
| 15 | [JSON Processing and Complex Data](PySpark/15_JSON_Processing_and_Complex_Data.ipynb) | Parsing JSON data and handling nested/complex data structures (arrays, structs, maps) |
| 16 | [Writing Data and Partitioning](PySpark/16_Writing_Data_and_Partitioning.ipynb) | Writing DataFrames to various formats and implementing partition strategies |
| 17 | [Spark Architecture and Deployment](PySpark/17_Spark_Architecture_and_Deployment.ipynb) | Deep dive into cluster managers, deployment modes (client/cluster), and resource allocation |
| 18 | [User Defined Functions (UDF)](PySpark/18_User_Defined_Functions_UDF.ipynb) | Creating custom UDFs and UDAFs for business-specific transformations |
| 19 | [Spark Execution Plan, DAG and Shuffle](PySpark/19_Spark_Execution_Plan_DAG_and_Shuffle.ipynb) | Understanding DAG visualization, stages, tasks, and shuffle operations |
| 20 | [Optimizing Shuffle in Spark](PySpark/20_Optimizing_Shuffle_in_Spark.ipynb) | Techniques to minimize expensive shuffle operations and improve performance |
| 21 | [Caching and Persistence](PySpark/21_Caching_and_Persistence.ipynb) | Using cache() and persist() effectively for iterative algorithms and reusable datasets |
| 22 | [Distributed Shared Variables - Broadcast, Accumulators](PySpark/22_Distributed_Shared_Variables_Broadcast_Accumulators.ipynb) | Using broadcast variables for lookup tables and accumulators for debugging |
| 23 | [Optimizing Joins in Spark](PySpark/23_Optimizing_Joins_in_Spark.ipynb) | Broadcast joins, sort-merge joins, and techniques to optimize join performance |
| 24 | [Dynamic Allocation and Resource Management](PySpark/24_Dynamic_Allocation_and_Resource_Management.ipynb) | Configuring dynamic resource allocation and managing executor memory/cores |
| 25 | [Skewness, Spillage and Salting](PySpark/25_Skewness_Spillage_and_Salting.ipynb) | Identifying and resolving data skew issues using salting and other techniques |
| 26 | [Adaptive Query Execution (AQE)](PySpark/26_Adaptive_Query_Execution_AQE.ipynb) | Leveraging AQE for runtime optimization, dynamic partition coalescing, and skew handling |
| 27 | [Spark SQL Catalog and Hints](PySpark/27_Spark_SQL_Catalog_and_Hints.ipynb) | Working with Spark catalog, metadata management, and using SQL hints |
| 28 | [Read and Write Cosmos DB](PySpark/28_Read_and_Write_Cosmos_DB.ipynb) | Integrating PySpark with Azure Cosmos DB for NoSQL data operations |
| 29 | [Delta Lake Basics - Time Travel, Schema Evolution](PySpark/29_Delta_Lake_Basics_Time_Travel_Schema_Evolution.ipynb) | Introduction to Delta Lake, ACID transactions, time travel, and schema evolution |
| 30 | [Data Scanning and Partitioning](PySpark/30_Data_Scanning_and_Partitioning.ipynb) | Understanding data scanning patterns and partition pruning for query optimization |
| 31 | [Delta Lake Optimization - ZOrder](PySpark/31_Delta_Lake_Optimization_ZOrder.ipynb) | Using OPTIMIZE and ZORDER BY commands to improve query performance on Delta tables |
| 32 | [Deletion Vectors and Liquid Clustering](PySpark/32_Deletion_Vectors_and_Liquid_Clustering.ipynb) | Advanced Delta Lake features for efficient deletes and automatic data clustering |
| 33 | [Spark Memory Management and OOM](PySpark/33_Spark_Memory_Management_and_OOM.ipynb) | Understanding memory allocation, troubleshooting OOM errors, and tuning memory settings |
| 34 | [Spark Connect](PySpark/34_Spark_Connect.ipynb) | Using Spark Connect for decoupled client-server architecture |
| 35 | [PySpark Unit Testing with Pytest](PySpark/35_PySpark_Unit_Testing_with_Pytest.ipynb) | Writing unit tests for PySpark applications using pytest framework |

---

## [Spark Streaming with PySpark](SparkStreamingWithPySpark/)
Learn real-time data processing with Apache Spark Structured Streaming and Kafka integration.

| Module | Topic | Description |
|--------|-------|-------------|
| 01 | [Spark Streaming Introduction](SparkStreamingWithPySpark/01_Spark_Streaming_Introduction.ipynb) | Overview of streaming concepts, use cases, and Spark Structured Streaming basics |
| 02 | [Spark Streaming Basics](SparkStreamingWithPySpark/02_Spark_Streaming_Basics.ipynb) | Understanding streaming queries, micro-batches, and continuous processing |
| 03 | [Environment Setup - Docker, Kafka](SparkStreamingWithPySpark/03_Environment_Setup_Docker_Kafka.ipynb) | Setting up Kafka and Spark streaming environment using Docker |
| 04 | [First Streaming Application - Sockets](SparkStreamingWithPySpark/04_First_Streaming_Application_Sockets.ipynb) | Building your first streaming application using socket source |
| 05 | [Streaming Output Modes and Internals](SparkStreamingWithPySpark/05_Streaming_Output_Modes_and_Internals.ipynb) | Understanding append, complete, and update output modes |
| 06 | [Streaming Architectures - Lambda vs Kappa](SparkStreamingWithPySpark/06_Streaming_Architectures_Lambda_vs_Kappa.ipynb) | Comparing Lambda and Kappa architectures for real-time data processing |
| 07 | [File Source and Checkpointing](SparkStreamingWithPySpark/07_File_Source_and_Checkpointing.ipynb) | Reading streaming data from files and implementing checkpointing for fault tolerance |
| 08 | [Checkpointing Deep Dive and Fault Tolerance](SparkStreamingWithPySpark/08_Checkpointing_Deep_Dive_and_Fault_Tolerance.ipynb) | Advanced checkpointing concepts and achieving exactly-once semantics |
| 09 | [Kafka Basics and Command Line](SparkStreamingWithPySpark/09_Kafka_Basics_and_Command_Line.ipynb) | Understanding Kafka architecture, topics, partitions, and CLI operations |
| 10 | [Reading from Kafka](SparkStreamingWithPySpark/10_Reading_from_Kafka.ipynb) | Consuming streaming data from Kafka topics in Spark |
| 11 | [Triggers and Performance Tuning](SparkStreamingWithPySpark/11_Triggers_and_Performance_Tuning.ipynb) | Configuring processing triggers and optimizing streaming query performance |
| 12 | [Writing to Multiple Sinks - JDBC](SparkStreamingWithPySpark/12_Writing_to_Multiple_Sinks_JDBC.ipynb) | Writing streaming data to multiple destinations including databases via JDBC |
| 13 | [Handling Errors and Exceptions](SparkStreamingWithPySpark/13_Handling_Errors_and_Exceptions.ipynb) | Error handling strategies, retry logic, and dead letter queues |
| 14 | [Event Time and Stateful Processing](SparkStreamingWithPySpark/14_Event_Time_and_Stateful_Processing.ipynb) | Working with event time vs processing time and maintaining state across batches |
| 15 | [Window Operations - Tumbling, Sliding, Session](SparkStreamingWithPySpark/15_Window_Operations_Tumbling_Sliding_Session.ipynb) | Implementing different window types for time-based aggregations |
| 16 | [Window Operations and Late Data Handling](SparkStreamingWithPySpark/16_Window_Operations_and_Late_Data_Handling.ipynb) | Managing watermarks and handling late-arriving data in streaming |
| 17 | [Azure Cosmos DB Integration](SparkStreamingWithPySpark/17_Azure_Cosmos_DB_Integration.ipynb) | Writing streaming data to Azure Cosmos DB for real-time analytics |

---

## [Data Warehousing](DataWarehousing/)
Comprehensive course on data warehousing concepts, dimensional modeling, star schema design, and slowly changing dimensions.

| Module | Topic | Description |
|--------|-------|-------------|
| 01 | [Introduction to Data Warehousing](DataWarehousing/01_Introduction_to_Data_Warehousing.ipynb) | Fundamentals of data warehousing, business intelligence, and analytics |
| 02 | [OLTP and Data Architecture](DataWarehousing/02_OLTP_and_Data_Architecture.ipynb) | Understanding OLTP systems, operational databases, and source system architecture |
| 03 | [OLAP and Analytical Reporting](DataWarehousing/03_OLAP_and_Analytical_Reporting.ipynb) | Introduction to OLAP concepts, cubes, and analytical query patterns |
| 04 | [Data Loading Strategies - ETL, ELT](DataWarehousing/04_Data_Loading_Strategies_ETL_ELT.ipynb) | Comparing ETL vs ELT approaches, batch vs incremental loading strategies |
| 05 | [Measures and Attributes - Data Modeling](DataWarehousing/05_Measures_and_Attributes_Data_Modeling.ipynb) | Identifying facts (measures) and dimensions (attributes) in business processes |
| 06 | [Grain and KPIs - Data Modeling](DataWarehousing/06_Grain_and_KPIs_Data_Modeling.ipynb) | Defining grain (level of detail) and key performance indicators for analytics |
| 07 | [Demo - Designing a Star Schema](DataWarehousing/07_Demo_Designing_a_Star_Schema.ipynb) | Hands-on demonstration of designing a star schema for a business scenario |
| 08 | [Conformed Dimensions and Bus Matrix](DataWarehousing/08_Conformed_Dimensions_and_Bus_Matrix.ipynb) | Creating reusable dimensions across fact tables and building dimensional bus matrix |
| 09 | [Advanced Dimension Types - Junk, Role-Playing, Degenerate](DataWarehousing/09_Advanced_Dimension_Types_Junk_RolePlaying_Degenerate.ipynb) | Special dimension patterns for complex modeling scenarios |
| 10 | [SCD Type 1 - Current State](DataWarehousing/10_SCD_Type_1_Current_State.ipynb) | Implementing Slowly Changing Dimension Type 1 (overwrite) strategy |
| 11 | [SCD Type 2 - Historical Data](DataWarehousing/11_SCD_Type_2_Historical_Data.ipynb) | Tracking full history with SCD Type 2 using effective dates and surrogate keys |
| 12 | [SCD Type 3 - Limited History](DataWarehousing/12_SCD_Type_3_Limited_History.ipynb) | Maintaining limited history using previous value columns |
| 13 | [Fact Tables - Transactional vs Aggregated](DataWarehousing/13_Fact_Tables_Transactional_vs_Aggregated.ipynb) | Designing transaction and periodic snapshot fact tables |
| 14 | [Fact Tables - Accumulating and Factless](DataWarehousing/14_Fact_Tables_Accumulating_and_Factless.ipynb) | Modeling pipeline/workflow processes and events without measures |
| 15 | [Star Schema vs Snowflake Schema](DataWarehousing/15_Star_Schema_vs_Snowflake_Schema.ipynb) | Comparing denormalized star schema vs normalized snowflake schema |
| 16 | [Advanced Issues - Early Facts, Late Dimensions](DataWarehousing/16_Advanced_Issues_Early_Facts_Late_Dimensions.ipynb) | Handling data timing issues when facts arrive before dimension records |

---

## [Databricks](Databricks/)
Complete Databricks zero-to-hero course covering lakehouse architecture, Unity Catalog, Delta Live Tables, and advanced features.

| Module | Topic | Description |
|--------|-------|-------------|
| 01 | [Databricks Zero To Hero - Introduction And Agenda](Databricks/01_Databricks_Zero_To_Hero_Introduction_And_Agenda.ipynb) | Course overview, learning path, and prerequisites for Databricks |
| 02 | [Why Databricks And Lakehouse Fundamentals](Databricks/02_Why_Databricks_And_Lakehouse_Fundamentals.ipynb) | Understanding lakehouse architecture and benefits of Databricks platform |
| 03 | [Databricks Architecture And Roles](Databricks/03_Databricks_Architecture_And_Roles.ipynb) | Control plane vs data plane, workspace architecture, and user roles |
| 04 | [Setup Azure Databricks Environment](Databricks/04_Setup_Azure_Databricks_Environment.ipynb) | Creating Azure Databricks workspace and configuring cloud resources |
| 05 | [Databricks Account Console Walkthrough](Databricks/05_Databricks_Account_Console_Walkthrough.ipynb) | Navigating account console, managing workspaces, users, and billing |
| 06 | [Databricks Workspace Walkthrough And First Run](Databricks/06_Databricks_Workspace_Walkthrough_And_First_Run.ipynb) | Exploring workspace UI, notebooks, clusters, and running first queries |
| 07 | [Databricks Internal Working With Azure](Databricks/07_Databricks_Internal_Working_With_Azure.ipynb) | Understanding how Databricks integrates with Azure services and networking |
| 08 | [Introduction To Unity Catalog And Governance](Databricks/08_Introduction_To_Unity_Catalog_And_Governance.ipynb) | Overview of Unity Catalog for data governance and centralized access control |
| 09 | [Legacy Hive Metastore - Managed vs External Tables](Databricks/09_Legacy_Hive_Metastore_Managed_vs_External_Tables.ipynb) | Working with Hive metastore and understanding table management options |
| 10 | [Setup Unity Catalog - Hands On](Databricks/10_Setup_Unity_Catalog_Hands_On.ipynb) | Step-by-step setup of Unity Catalog metastore and storage credentials |
| 11 | [Unity Catalog - Create Catalogs And External Locations](Databricks/11_Unity_Catalog_Create_Catalogs_And_External_Locations.ipynb) | Creating catalogs and configuring external storage locations |
| 12 | [Unity Catalog - Create Schemas And Tables With Locations](Databricks/12_Unity_Catalog_Create_Schemas_And_Tables_With_Locations.ipynb) | Organizing data with schemas and tables in Unity Catalog |
| 13 | [Unity Catalog - Managed vs External Tables Deep Dive](Databricks/13_Unity_Catalog_Managed_vs_External_Tables_Deep_Dive.ipynb) | Understanding lifecycle and data ownership differences between table types |
| 14 | [Delta Tables - Cloning, CTAS and Views](Databricks/14_Delta_Tables_Cloning_CTAS_and_Views.ipynb) | Creating tables using CTAS, shallow/deep cloning, and managing views |
| 15 | [Delta Lake Merge Operations](Databricks/15_Delta_Lake_Merge_Operations.ipynb) | Implementing MERGE for upserts and handling change data capture |
| 16 | [Deletion Vectors And Liquid Clustering](Databricks/16_Deletion_Vectors_And_Liquid_Clustering.ipynb) | Using deletion vectors for efficient deletes and liquid clustering for optimization |
| 17 | [Unity Catalog Volumes](Databricks/17_Unity_Catalog_Volumes.ipynb) | Managing unstructured data (files) using Unity Catalog volumes |
| 18 | [Databricks Utilities Deep Dive](Databricks/18_Databricks_Utilities_Deep_Dive.ipynb) | Using dbutils for file system operations, secrets, and widgets |
| 19 | [Parameterize Notebooks and Workflows](Databricks/19_Parameterize_Notebooks_and_Workflows.ipynb) | Creating reusable notebooks with parameters and passing values between notebooks |
| 20 | [Databricks Compute Architecture](Databricks/20_Databricks_Compute_Architecture.ipynb) | Understanding cluster types, runtimes, and compute configurations |
| 21 | [Databricks Cluster Policies And Instance Pools](Databricks/21_Databricks_Cluster_Policies_And_Instance_Pools.ipynb) | Enforcing standards with policies and optimizing costs with instance pools |
| 22 | [Databricks Workflows And Orchestration](Databricks/22_Databricks_Workflows_And_Orchestration.ipynb) | Building data pipelines with Databricks Workflows (formerly Jobs) |
| 23 | [Data Ingestion With Copy Into](Databricks/23_Data_Ingestion_With_Copy_Into.ipynb) | Efficiently loading data using COPY INTO command with idempotent loads |
| 24 | [Databricks Auto Loader And Schema Evolution](Databricks/24_Databricks_Auto_Loader_And_Schema_Evolution.ipynb) | Incrementally ingesting files with Auto Loader and handling schema changes |
| 25 | [Medallion Architecture Explained](Databricks/25_Medallion_Architecture_Explained.ipynb) | Implementing Bronze, Silver, and Gold layers for data quality and refinement |
| 26 | [Delta Live Tables Introduction](Databricks/26_Delta_Live_Tables_Introduction.ipynb) | Declarative ETL framework for building reliable data pipelines |
| 27 | [DLT - Incremental Processing And Evolution](Databricks/27_DLT_Incremental_Processing_And_Evolution.ipynb) | Implementing incremental processing and handling schema evolution in DLT |
| 28 | [DLT - Autoloader, Append Flow And Dynamic Tables](Databricks/28_DLT_Autoloader_Append_Flow_And_Dynamic_Tables.ipynb) | Using Auto Loader in DLT, append-only flows, and dynamic table patterns |
| 29 | [DLT - Apply Changes SCD1, SCD2](Databricks/29_DLT_Apply_Changes_SCD1_SCD2.ipynb) | Implementing slowly changing dimensions using APPLY CHANGES API |
| 30 | [Manage Data Quality With DLT](Databricks/30_Manage_Data_Quality_With_DLT.ipynb) | Defining expectations, constraints, and quarantine rules for data quality |
| 31 | [DLT Advanced - SkipCommits, FullRefresh and Workflows](Databricks/31_DLT_Advanced_SkipCommits_FullRefresh_and_Workflows.ipynb) | Advanced DLT features and integrating with Databricks Workflows |
| 32 | [Secret Management In Databricks](Databricks/32_Secret_Management_In_Databricks.ipynb) | Securely storing credentials using Databricks secrets and scopes |
| 33 | [User Management In Databricks](Databricks/33_User_Management_In_Databricks.ipynb) | Managing users, service principals, and groups in Databricks |
| 34 | [Manage Privileges In Unity Catalog](Databricks/34_Manage_Privileges_In_Unity_Catalog.ipynb) | Implementing RBAC with GRANT/REVOKE and understanding privilege inheritance |
| 35 | [Databricks Functions - Scalar And Table](Databricks/35_Databricks_Functions_Scalar_And_Table.ipynb) | Creating reusable SQL and Python functions in Unity Catalog |
| 36 | [Row Level Filters In Unity Catalog](Databricks/36_Row_Level_Filters_In_Unity_Catalog.ipynb) | Implementing row-level security to restrict data access by user |
| 37 | [Column Level Masking In Databricks](Databricks/37_Column_Level_Masking_In_Databricks.ipynb) | Applying dynamic column masking for sensitive data protection |
| 38 | [Workspace Catalog Binding In Unity Catalog](Databricks/38_Workspace_Catalog_Binding_In_Unity_Catalog.ipynb) | Controlling catalog visibility and access across workspaces |
| 39 | [Delta Sharing - Databricks To Databricks](Databricks/39_Delta_Sharing_Databricks_To_Databricks.ipynb) | Securely sharing data across Databricks accounts and organizations |
| 40 | [Enable Serverless Compute](Databricks/40_Enable_Serverless_Compute.ipynb) | Using serverless SQL warehouses and compute for instant startup |
| 41 | [Databricks SQL And Warehouses](Databricks/41_Databricks_SQL_And_Warehouses.ipynb) | Running SQL queries, managing warehouses, and query history |
| 42 | [Streaming Tables And Materialized Views - DBSQL](Databricks/42_Streaming_Tables_And_Materialized_Views_DBSQL.ipynb) | Creating streaming tables and materialized views in Databricks SQL |
| 43 | [Lakehouse Federation - External Data Querying](Databricks/43_Lakehouse_Federation_External_Data_Querying.ipynb) | Querying external data sources (PostgreSQL, MySQL, etc.) without ETL |
| 44 | [Databricks SQL Alerts](Databricks/44_Databricks_SQL_Alerts.ipynb) | Setting up alerts and notifications for data monitoring |
| 45 | [Unity Catalog Metric Views](Databricks/45_Unity_Catalog_Metric_Views.ipynb) | Creating semantic layer with metrics for consistent business definitions |
| 46 | [AI BI Dashboards And Visualization](Databricks/46_AI_BI_Dashboards_And_Visualization.ipynb) | Building interactive dashboards and visualizations in Databricks |
| 47 | [AI BI Genie - Natural Language Querying](Databricks/47_AI_BI_Genie_Natural_Language_Querying.ipynb) | Using AI-powered natural language interface to query data |
| 48 | [Databricks Git Integration Setup](Databricks/48_Databricks_Git_Integration_Setup.ipynb) | Connecting notebooks to Git repositories for version control |
| 49 | [Databricks CLI Installation And Authentication](Databricks/49_Databricks_CLI_Installation_And_Authentication.ipynb) | Installing and configuring Databricks CLI for automation |
| 50 | [Databricks Asset Bundles Part1 - Configuration and Dev Deployment](Databricks/50_Databricks_Asset_Bundles_Part1_Configuration_and_Dev_Deployment.ipynb) | Introduction to DABs for Infrastructure-as-Code deployments |
| 51 | [Databricks Asset Bundles Part2 - CI/CD with Azure DevOps](Databricks/51_Databricks_Asset_Bundles_Part2_CICD_with_Azure_DevOps.ipynb) | Implementing CI/CD pipelines for Databricks using Azure DevOps |

---

# Projects

## [Data Warehousing with PySpark](Projects/DataWarehousingWithPySpark/)
End-to-end data warehousing project implementing dimensional modeling, ETL pipelines, and SCD handling with PySpark.

| Module | Topic | Description |
|--------|-------|-------------|
| 01 | [Data Warehousing PySpark Project Overview](Projects/DataWarehousingWithPySpark/01_Data_Warehousing_PySpark_Project_Overview.ipynb) | Project introduction, business requirements, and architecture overview |
| 02 | [Architecture and Data Model Design](Projects/DataWarehousingWithPySpark/02_Architecture_and_Data_Model_Design.ipynb) | Designing star schema, identifying dimensions and facts, and data flow |
| 03 | [Environment Setup - Docker, PySpark](Projects/DataWarehousingWithPySpark/03_Environment_Setup_Docker_PySpark.ipynb) | Setting up local development environment with Docker and PySpark |
| 04 | [S3, IAM and Spark Configuration](Projects/DataWarehousingWithPySpark/04_S3_IAM_and_Spark_Configuration.ipynb) | Configuring AWS S3 storage, IAM roles, and Spark connectors |
| 05 | [ETL Architecture and Loading Strategy](Projects/DataWarehousingWithPySpark/05_ETL_Architecture_and_Loading_Strategy.ipynb) | Designing landing, staging, and dimension/fact loading patterns |
| 06 | [Initialize Database and Tables](Projects/DataWarehousingWithPySpark/06_Initialize_Database_and_Tables.ipynb) | Creating database schemas and initializing dimension and fact tables |
| 07 | [Utility Functions Setup](Projects/DataWarehousingWithPySpark/07_Utility_Functions_Setup.ipynb) | Building reusable utility functions for ETL processes |
| 08 | [Date Landing Load](Projects/DataWarehousingWithPySpark/08_Date_Landing_Load.ipynb) | Loading date data from source to landing zone |
| 09 | [Date Staging Load](Projects/DataWarehousingWithPySpark/09_Date_Staging_Load.ipynb) | Transforming and loading date data to staging area |
| 10 | [Date Dimension Load](Projects/DataWarehousingWithPySpark/10_Date_Dimension_Load.ipynb) | Populating date dimension with calendar attributes and hierarchies |
| 11 | [Athena Integration - Symlink Manifest](Projects/DataWarehousingWithPySpark/11_Athena_Integration_Symlink_Manifest.ipynb) | Enabling AWS Athena queries using symlink manifest files |
| 12 | [Store Dimension Load - End to End](Projects/DataWarehousingWithPySpark/12_Store_Dimension_Load_End_to_End.ipynb) | Complete ETL pipeline for store dimension from source to target |
| 13 | [Customer Dimension Load - SCD2](Projects/DataWarehousingWithPySpark/13_Customer_Dimension_Load_SCD2.ipynb) | Implementing SCD Type 2 for tracking customer history changes |
| 14 | [Product Dimension Load - SCD2](Projects/DataWarehousingWithPySpark/14_Product_Dimension_Load_SCD2.ipynb) | Implementing SCD Type 2 for product dimension with effective dating |
| 15 | [Sales Fact Landing Load](Projects/DataWarehousingWithPySpark/15_Sales_Fact_Landing_Load.ipynb) | Ingesting transactional sales data into landing zone |
| 16 | [Sales Fact Staging Load](Projects/DataWarehousingWithPySpark/16_Sales_Fact_Staging_Load.ipynb) | Cleansing and transforming sales transactions in staging |
| 17 | [Sales Fact Dimension Load](Projects/DataWarehousingWithPySpark/17_Sales_Fact_Dimension_Load.ipynb) | Loading fact table with surrogate key lookups and measures |
| 18 | [Incremental Load - Spark Submit](Projects/DataWarehousingWithPySpark/18_Incremental_Load_Spark_Submit.ipynb) | Implementing incremental loading using Spark submit for automation |
| 19 | [Analytical Queries - Athena](Projects/DataWarehousingWithPySpark/19_Analytical_Queries_Athena.ipynb) | Running business intelligence queries on the data warehouse using Athena |

---

## How to Use

1. Clone this repository
2. Navigate to the desired course folder
3. Open the Jupyter notebooks in sequence
4. Each notebook contains explanations, code examples, and hands-on exercises

## Prerequisites

- Python 3.8+
- Apache Spark 3.x
- Docker (for environment setup)
- Jupyter Notebook or JupyterLab
- AWS account (for some projects)
- Azure account (for Databricks courses)

---

**Happy Learning!**
