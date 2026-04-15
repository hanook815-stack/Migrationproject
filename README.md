# Migration project
Eend-to-end data migration project where we moved data from on-prem SQL Server to Azure Data Lake using Azure Data Factory. 
The solution is metadata-driven and supports incremental loading using a watermark mechanism.

# Architectur
============
Source (SQL Server)
      ↓
ADF Pipeline
      ↓
ADLS (Parquet files)
      ↓
Watermark Table (control)

# Source System
==================
On-prem SQL Server
Multiple tables (DimCustomer, FactSales, etc.)

I created a watermark table to track incremental loads.
Columns:
TableSchema
TableName
WatermarkColumn
LastLoadValue

🔄 4. Pipeline Flow (Step-by-Step)

🔹 Step 1: Lookup Activity
“First, I used a Lookup activity to fetch metadata from the watermark table.”

🔹 Step 2: ForEach Activity
“Then I used a ForEach activity to loop through each table dynamically.”

🔹 Step 3: Lookup MAX Value
“Inside the loop, I used another Lookup to get the latest MAX value from the source table.”

🔹 Step 4: IF Condition (Core Logic)
“Then I used an IF condition to compare the source MAX value with the last loaded value.”

🔹 Step 5: Copy Activity
“If new data is available, I used Copy Activity to load data into ADLS in Parquet format.”

🔹 Step 6: Stored Procedure
“After successful load, I executed a stored procedure to update the watermark table.”

# Data Storage (ADLS)
Data is stored in ADLS Gen2 in Parquet format with dynamic partitioning.
Example : raw/TableName/yyyy/MM/dd/file.parquet

**Key Features**
✔ Metadata-driven pipeline
✔ Incremental loading (watermark logic)
✔ Dynamic table handling
✔ Cost optimization (no full load every time)
✔ Scalable design using ForEach


**Lookup activity** is to get all the tables from SQL server database. Get metadata (list of tables)

  Parameters:
  TableSchema = @item().TableSchema
  TableName = @item().TableName
  WatermarkColumn = @item().WatermarkColumn
  LastLoadValue = @item().LastLoadValue
  
  <img width="756" height="371" alt="image" src="https://github.com/user-attachments/assets/3f64d00c-6ed7-4725-a472-175ab44945af" />
  
<img width="893" height="524" alt="image" src="https://github.com/user-attachments/assets/205f7f0e-2bf9-4c00-a2ac-d054eee351de" />

inlookupactivity under setting select SQL serever and install ingreation runtime linked services.
then add data properties
TableSchema = @pipeline().parameters.TableSchema
TableName = @pipeline().parameters.TableName

Then **ForEach** is used to process multiple items (tables/files/records) dynamically in a loop
<img width="818" height="389" alt="image" src="https://github.com/user-attachments/assets/41cc242a-64f6-413f-8e72-263bc216799e" />

  Under setting -- > item = @activity('LKUP_WATER_MARK').output.value

  Then add **lookupactivity** **inside ForEach activity** to Get latest MAX value from source table
  <img width="800" height="552" alt="image" src="https://github.com/user-attachments/assets/74ecec55-e12d-4220-8fc4-213c0b3b9e88" />

      Under setting add Dataset Properties
         TableSchema = @pipeline().parameters.TableSchema
        TableName = @pipeline().parameters.TableName

        Add under query
        @concat(
    'SELECT MAX(', item().WatermarkColumn, ') AS MaxValue FROM ',
    item().TableSchema, '.', item().TableName
)


Next add If condition ---- IF condition decides whether to load data or skip
 **inside if condition** acitivity add **copy activity and Procedure activity**
<img width="816" height="487" alt="image" src="https://github.com/user-attachments/assets/65ea5173-8df1-4005-a37d-12eec1ecfefe" />

 Copy acivity add below parameters under source
 <img width="871" height="489" alt="image" src="https://github.com/user-attachments/assets/0da02dbf-7881-4a1e-a9be-85b621701eca" />
 <img width="758" height="328" alt="image" src="https://github.com/user-attachments/assets/2dae6340-775e-49bf-811e-cb93d7d2088e" />

           TableSchema = @pipeline().parameters.TableSchema
          TableName = @pipeline().parameters.TableName

          **Under Query :**   
          @concat(
    'SELECT * FROM ',
    item().TableSchema, '.', item().TableName,
    if(
        equals(item().LastLoadValue, null),
        '',
        concat(
            ' WHERE ', item().WatermarkColumn,
            ' > ''', item().LastLoadValue, ''''
        )
    )
)

Add dataset parameters TableName = @item().TableName

Now crate Store procedure and link with Copy activity -- > procedure ( We execute them only when new data is available) 
<img width="788" height="477" alt="image" src="https://github.com/user-attachments/assets/75743385-8aca-4131-80cd-619a7c424400" />

select store procedure name -- updatewatermark table
Copy activity -- Only runs when new data availabel
Store procedure -- if new data exit and true then execute copy activity and procedure acitivity

add store procedure parameters as per the store procedure in SQL server

TableName = @item().TableName
TableSchema = @item().TableSchema
WatermarkColumn = @item().WatermarkColumn

**PIPELINE**


<img width="815" height="375" alt="image" src="https://github.com/user-attachments/assets/33f217a9-ee1b-4426-83bc-cb8b0a94ea3b" />


**Simple Diagram**

        ┌──────────────────────┐
        │  SQL Server (Source) │
        └─────────┬────────────┘
                  │
                  ▼
        ┌──────────────────────┐
        │  Lookup (Watermark)  │
        └─────────┬────────────┘
                  │
                  ▼
        ┌──────────────────────┐
        │      ForEach Loop     │
        └─────────┬────────────┘
                  │
        ┌─────────▼───────────┐
        │   Lookup MAX Value  │
        └─────────┬───────────┘
                  │
        ┌─────────▼───────────┐
        │    IF Condition     │
        └───────┬─────┬───────┘
                │     │
           FALSE│     │TRUE
                │     ▼
                │  ┌──────────────┐
                │  │ Copy Activity│
                │  └──────┬───────┘
                │         ▼
                │  ┌──────────────┐
                │  │ Stored Proc  │
                │  └──────┬───────┘
                │         ▼
                │   Update Watermark
                │
                ▼
            Skip Load

                  ▼
        ┌──────────────────────┐
        │ ADLS (Parquet Files) │
        └──────────────────────┘
