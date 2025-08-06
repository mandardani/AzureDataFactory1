## Work Flow Diagram

Start
  |
  |  (ADF Pipeline Execution)
  v
+--------------------------------+
|      Lookup Activity           |
|      (lookup_emr_config)       |
|      - Reads load_config.csv   |
+--------------------------------+
  |
  |  (Output: list of tables to load)
  v
+--------------------------------+
|       ForEach Activity         |
|      (sequential_load)         |
|      - Iterates over each item |
|        from lookup output      |
+--------------------------------+
  |
  |  (For each item/table)
  v
+-------------------------------------------------------------+
|    Generate Target File Path                                |
|    (e.g., bronze/tablename/...)                             |
+-------------------------------------------------------------+
  |
  v
+-------------------------------------------------------------+
|    Condition: Check if file already exists at target location |
+-------------------------------------------------------------+
  |
  +---( If TRUE )---> +-------------------------------------+
  |                   |          Archive File Activity      |
  |                   |          - Moves existing file to   |
  |                   |            archive folder (e.g.,    |
  |                   |            bronze/tablename/archive)|
  |                   +-------------------------------------+
  |                                   |
  |                                   v
  +---( If FALSE )---> +-------------------------------------+
                      |    Condition: Is Load Type 'Full'?   |
                      |    (Check @item().loadtype)          |
                      +-------------------------------------+
                                 |
+-----( If TRUE )----------------+---------------------------( If FALSE )--------+
|                                |                           |                   |
|                                v                           v                   v
|                 +----------------------------+  +--------------------------------+
|                 |    Copy Data Activity      |  |     Lookup last run time       |
|                 |    (Full Load)             |  |     - From load_logs table     |
|                 |    - Source: SQL Table     |  |       (on Databricks)          |
|                 |    - Sink: Parquet file in |  |                                |
|                 |      ADLS Gen2 bronze      |  +--------------------------------+
|                 +----------------------------+                          |
|                                |                                         v
|                                v                           +---------------------------------+
|                 +----------------------------+             |    Copy Data Activity           |
|                 |   Log Job Run Activity     |             |    (Incremental Load)           |
|                 |   - Writes pipeline status |             |    - Source: SQL Table with     |
|                 |   to load_logs table in    |             |     watermark filter (@item().  |
|                 |   Databricks/Unity Catalog |             |    watermarkcolumn > last run)  |
|                 +----------------------------+             |    - Sink: Parquet file in      |
|                                |                           |      ADLS Gen2 bronze           |
+--------------------------------+                           +---------------------------------+
                 |                                                              |
                 v                                                              v
   (Continue with next item)                         +----------------------------------+
                                                      |   Log Job Run Activity         |
                                                      |   - Writes pipeline status and |
                                                      |     new watermark value to     |
                                                      |     load_logs table            |
                                                     +----------------------------------+
                                                                     |
                                                                     v
                                                            (Continue with next item)

External Components:
•	Data Sources: Hospital1 and Hospital10 (Azure SQL Databases)
•	Data Lake: Azure Data Lake Storage Gen2 (config container, bronze container, archive subfolders)
•	Configuration File: load_config.csv (inside the config container)
•	Log Storage: Databricks Unity Catalog (pipelines.jobrun.load_logs table)
•	Security: Azure Key Vault (stores database passwords)

