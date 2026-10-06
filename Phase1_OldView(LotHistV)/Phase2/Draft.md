To achieve this multi-step workflow—where the agent first generates a SQL query, then retrieves the database results, and finally interprets those results using your .docx file—you need to carefully structure the agent's logic.
Because Microsoft Copilot Studio orchestrates actions dynamically, relying only on a single prompt in the Overview tab can cause the agent to confuse the order of operations. It might try to read the document before running the query, or skip the document entirely.
Here is the exact setup and step-by-step instructions to ensure the agent executes this sequence flawlessly.
------------------------------
## Step 1: Upload and Tag the Document Properly
Before editing the prompt, give the orchestrator a highly specific semantic hook for your file.
 
   1. Go to the Knowledge tab in Copilot Studio and upload your .docx file.
   2. Set the Name to: Database_Column_Dictionary
   3. Set the Description to: Contains the business definitions, logic, and meaning behind the values for the 6 columns returned by the production database SQL queries.
 
------------------------------
## Step 2: Use an Exact, Multi-Phase System Instruction
Replace or update your agent's Instructions on the Overview page. You must explicitly break the agent's thinking down into phases so it doesn't skip steps.
Copy and paste the following prompt structure into the Overview instructions text box:
 
# Role & Core Objective
You are a data analyst assistant. Your job is to translate user natural language questions into SQL queries, showcase the database results, and provide a clear business interpretation using your connected knowledge base.
 
# Execution Order (Strict Rule)
You must follow these steps in exact sequential order for every user request:
1. PHASE 1: Write the SQL query based on the user's question.
2. PHASE 2: Execute the query against the production database endpoint.
3. PHASE 3: Read the raw data output from Phase 2. Open and reference the connected [Database_Column_Dictionary] (type '/' to select your uploaded docx here) to map the raw values and codes to their actual business meanings.
4. PHASE 4: Generate the final response using the exact Output Format.
 
# Output Format
Your final response to the user must strictly use this markdown format:
 
**Query:** 
[Insert the exact SQL query generated here]
 
**Result:** 
[Insert the raw data or table results returned from the database execution here]
 
**Business Interpretation:** 
[Combine the raw data from the 'Result' section with the column definitions found in the connected [Database_Column_Dictionary] document. Translate codes, statuses, or numerical boundaries into plain business language to explain what this data actually means for the user's business context.]
 
# Guardrails
- Do not attempt to write the Business Interpretation without cross-referencing the connected document.
- If a column value is returned that is not defined in the document, state the raw value but note that a business definition is missing.
 
(Note: In the prompt above, delete [Database_Column_Dictionary] where noted and press the / key on your keyboard to select the actual uploaded document tile so it turns into a hard link).
------------------------------
## Step 3: Configure the Generative Answers Node (Crucial)
By default, Copilot Studio might try to use the .docx file to answer the user's initial question instead of using SQL. To prevent this, ensure your topics are routed correctly:
 
   1. Ensure your SQL query action (e.g., a Power Automate flow or a connector plugin) is triggered first in your conversational topic.
   2. After the SQL action node returns the data to a variable (e.g., Topic.QueryResult), insert a Create Generative Answers node.
   3. In that Generative Answers node, set the Input to a combined string, such as:
   "Interpret these database results: " & Topic.QueryResult & " based on the user's original question: " & System.LastMessage.Text
   4. Under the Knowledge settings inside that specific node, explicitly select your uploaded .docx file as the sole data source. This forces the LLM to use the document only at the end of the chain to generate the "Business Interpretation" text.
 
To ensure this runs smoothly, could you share:
 
* How the SQL query is executed? (e.g., Are you using a Power Automate flow, a custom plugin/connector, or standard Copilot Studio topics?)
* What are a few examples of the raw values or codes in the 6 columns that the .docx file translates? (e.g., Status 1 means Active, or Code A means High Priority?)
 
To achieve this multi-step workflow—where the agent first generates a SQL query, then retrieves the database results, and finally interprets those results using your .docx file—you need to carefully structure the agent's logic.
Because Microsoft Copilot Studio orchestrates actions dynamically, relying only on a single prompt in the Overview tab can cause the agent to confuse the order of operations. It might try to read the document before running the query, or skip the document entirely.
Here is the exact setup and step-by-step instructions to ensure the agent executes this sequence flawlessly.
------------------------------
## Step 1: Upload and Tag the Document Properly
Before editing the prompt, give the orchestrator a highly specific semantic hook for your file.
 
   1. Go to the Knowledge tab in Copilot Studio and upload your .docx file.
   2. Set the Name to: Database_Column_Dictionary
   3. Set the Description to: Contains the business definitions, logic, and meaning behind the values for the 6 columns returned by the production database SQL queries.
 
------------------------------
## Step 2: Use an Exact, Multi-Phase System Instruction
Replace or update your agent's Instructions on the Overview page. You must explicitly break the agent's thinking down into phases so it doesn't skip steps.
Copy and paste the following prompt structure into the Overview instructions text box:
 
# Role & Core Objective
You are a data analyst assistant. Your job is to translate user natural language questions into SQL queries, showcase the database results, and provide a clear business interpretation using your connected knowledge base.
 
# Execution Order (Strict Rule)
You must follow these steps in exact sequential order for every user request:
1. PHASE 1: Write the SQL query based on the user's question.
2. PHASE 2: Execute the query against the production database endpoint.
3. PHASE 3: Read the raw data output from Phase 2. Open and reference the connected [Database_Column_Dictionary] (type '/' to select your uploaded docx here) to map the raw values and codes to their actual business meanings.
4. PHASE 4: Generate the final response using the exact Output Format.
 
# Output Format
Your final response to the user must strictly use this markdown format:
 
**Query:** 
[Insert the exact SQL query generated here]
 
**Result:** 
[Insert the raw data or table results returned from the database execution here]
 
**Business Interpretation:** 
[Combine the raw data from the 'Result' section with the column definitions found in the connected [Database_Column_Dictionary] document. Translate codes, statuses, or numerical boundaries into plain business language to explain what this data actually means for the user's business context.]
 
# Guardrails
- Do not attempt to write the Business Interpretation without cross-referencing the connected document.
- If a column value is returned that is not defined in the document, state the raw value but note that a business definition is missing.
 
(Note: In the prompt above, delete [Database_Column_Dictionary] where noted and press the / key on your keyboard to select the actual uploaded document tile so it turns into a hard link).
------------------------------
## Step 3: Configure the Generative Answers Node (Crucial)
By default, Copilot Studio might try to use the .docx file to answer the user's initial question instead of using SQL. To prevent this, ensure your topics are routed correctly:
 
   1. Ensure your SQL query action (e.g., a Power Automate flow or a connector plugin) is triggered first in your conversational topic.
   2. After the SQL action node returns the data to a variable (e.g., Topic.QueryResult), insert a Create Generative Answers node.
   3. In that Generative Answers node, set the Input to a combined string, such as:
   "Interpret these database results: " & Topic.QueryResult & " based on the user's original question: " & System.LastMessage.Text
   4. Under the Knowledge settings inside that specific node, explicitly select your uploaded .docx file as the sole data source. This forces the LLM to use the document only at the end of the chain to generate the "Business Interpretation" text.
 
⁠To ensure this runs smoothly, could you share:
 
* How the SQL query is executed? (e.g., Are you using a Power Automate flow, a custom plugin/connector, or standard Copilot Studio topics?)
* What are a few examples of the raw values or codes in the 6 columns that the .docx file translates? (e.g., Status 1 means Active, or Code A means High Priority?)
 


# Hist_Rec analysis code 

```sql
WITH ordered AS (
    SELECT
        LOT,
        DATE_TIME,
        USERNAME,
        HISTCODE,
        HIS_REC,
 
        LAG(USERNAME) OVER (
            PARTITION BY LOT
            ORDER BY DATE_TIME
        ) AS prev_username
    FROM TrainingVision.LotHistv
    WHERE LOT = 'YOUR_LOT'
),
marked AS (
    SELECT *,
        CASE
            WHEN prev_username = USERNAME THEN 0
            ELSE 1
        END AS new_group
    FROM ordered
),
numbered AS (
    SELECT *,
        SUM(new_group) OVER (
            PARTITION BY LOT
            ORDER BY DATE_TIME
            ROWS UNBOUNDED PRECEDING
        ) AS sequence
    FROM marked
)
SELECT
    sequence,
    LOT,
    USERNAME,
    HISTCODE,
    STRING_AGG(HIS_REC, ' ')
        WITHIN GROUP (ORDER BY DATE_TIME) AS HIS_REC
FROM numbered
GROUP BY
    LOT,
    sequence,
    USERNAME,
    HISTCODE
ORDER BY
    sequence;
```




Ran in fabric the working code was 

```sql


WITH ordered AS (
    SELECT
        LOT,
        DATE_TIME,
        USERNAME,
        HISTCODE,
        HIST_REC,
        LAG(USERNAME) OVER (
            PARTITION BY LOT
            ORDER BY DATE_TIME
        ) AS prev_username
    FROM TrainingVision.LotHistV
    WHERE LOT = '5125A004B'
),
marked AS (
    SELECT *,
        CASE
            WHEN prev_username = USERNAME THEN 0
            ELSE 1
        END AS new_group
    FROM ordered
),
numbered AS (
    SELECT *,
        SUM(new_group) OVER (
            PARTITION BY LOT
            ORDER BY DATE_TIME
            ROWS UNBOUNDED PRECEDING
        ) AS sequence
    FROM marked
)
SELECT
    sequence,
    LOT,
    USERNAME,
    STRING_AGG(HIST_REC, ' ')
        WITHIN GROUP (ORDER BY DATE_TIME) AS HIST_REC
FROM numbered
GROUP BY
    sequence,
    LOT,
    USERNAME
ORDER BY
    sequence;

```
