# mysqlPracticeQuestions

# 📌 SQL Practice Project — Revenue Leakage Detection

## 📖 Problem Statement

### Revenue Leakage

A company wants to identify contracts where clients were **billed less than the contracted amount for 3 or more consecutive months**.

The goal is to find:

- Contract details
- Client details
- Number of consecutive leakage months
- Total revenue leakage amount
- First leakage month
- Last leakage month

---

# 🏗️ Database Schema

## Contracts Table

Stores contract information.

<details>
<summary>View Table Creation Query</summary>

```sql
CREATE TABLE contracts (
    contract_id INT PRIMARY KEY,
    client_id INT,
    contracted_amount INT,
    start_date DATE,
    end_date DATE
);
```

</details>


## Invoices Table

Stores monthly invoice information.

<details>
<summary>View Table Creation Query</summary>

```sql
CREATE TABLE invoices (
    invoice_id INT PRIMARY KEY,
    contract_id INT,
    invoice_amount INT,
    invoice_date DATE
);
```

</details>


---

# 📊 Sample Data

<details>
<summary>Insert Contract Data</summary>

```sql
INSERT INTO contracts VALUES

(1,101,10000,'2025-01-01','2025-08-01'),
(2,102,15000,'2025-01-01','2025-10-01'),
(3,103,12000,'2025-01-01','2025-06-01'),
(4,104,20000,'2025-01-01','2025-09-01');

```

</details>


<details>
<summary>Insert Invoice Data</summary>

```sql
INSERT INTO invoices VALUES

(1,1,10000,'2025-01-01'),
(2,1,8000,'2025-02-01'),
(3,1,7000,'2025-03-01'),
(4,1,6000,'2025-04-01'),
(5,1,10000,'2025-05-01'),
(6,1,5000,'2025-06-01'),
(7,1,4000,'2025-07-01'),
(8,1,10000,'2025-08-01');


-- Contract 2
(9,2,15000,'2025-01-01'),
(10,2,12000,'2025-02-01'),
(11,2,10000,'2025-03-01'),
(12,2,15000,'2025-04-01'),
(13,2,9000,'2025-05-01'),
(14,2,8000,'2025-06-01'),
(15,2,7000,'2025-07-01');


-- Contract 3 (No leakage)
(19,3,12000,'2025-01-01'),
(20,3,12000,'2025-02-01'),
(21,3,12000,'2025-03-01'),
(22,3,12000,'2025-04-01'),
(23,3,12000,'2025-05-01'),
(24,3,12000,'2025-06-01'),


-- Contract 4 (missing invoice months scenario)
(25,4,20000,'2025-01-01'),
(26,4,15000,'2025-02-01'),
(27,4,12000,'2025-05-01'),
(28,4,20000,'2025-06-01'),
(29,4,10000,'2025-07-01'),
(30,4,8000,'2025-08-01'),
(31,4,20000,'2025-09-01');


```

</details>


---

# 🎯 Expected Output

| Column | Description |
|---|---|
| contract_id | Contract identifier |
| client_id | Client identifier |
| consecutive_months | Number of continuous leakage months |
| total_leakage | Total amount under billed |
| first_leakage_date | Starting leakage month |
| last_leakage_date | Ending leakage month |


---

# 🧠 Approach

### Step 1: Join Contracts and Invoices

Combine contract details with billing information.

### Step 2: Identify Leakage

Filter records where:

```
contracted_amount > invoice_amount
```

### Step 3: Remove Duplicate Months

Keep only one record per client-month.

### Step 4: Find Consecutive Months

Use:

- ROW_NUMBER()
- DATE_SUB()
- Window Functions

Logic:

```
month - row_number = same group
```

Example:

| Month | Row Number | Month - RN |
|-|-|-|
| Jan | 1 | Dec |
| Feb | 2 | Dec |
| Mar | 3 | Dec |

Same result means consecutive months.


---

# 💻 MySQL Solution Query

<details>
<summary>Click to View SQL Query</summary>

```sql


select * from contracts;
select * from invoices;

with join_contract_invoice_table as (
	select 
		invoices.invoice_id, 
        contracts.contract_id,
        contracts.client_id, 
        contracts.contracted_amount,
        invoices.invoice_amount,
        invoices.invoice_date,
        date_format(invoices.invoice_date, '%Y-%m-01') as normalized_invoice_date
    from invoices
    join contracts 
    on invoices.contract_id = contracts.contract_id
),
entries_with_less_invoice_amount as (
	
	select * from join_contract_invoice_table where contracted_amount > invoice_amount
),
duplicate_check as (
	select *, row_number() over(partition by client_id, normalized_invoice_date) as unique_id from entries_with_less_invoice_amount
),
unique_months as (
	select * from duplicate_check where unique_id = 1
),
ranking_months as (
	select *, dense_rank() over(partition by contract_id order by normalized_invoice_date) rn from unique_months
),
consecutive_months as(
	select *, date_sub(normalized_invoice_date, interval rn month) as consecutive from ranking_months
),
grouping_consecutive_months as (
    select *, count(*)  over(partition by client_id, consecutive) as total_consecutive_months from consecutive_months
)

select client_id, contract_id, contracted_amount,  invoice_id, invoice_date, invoice_amount, total_consecutive_months 
from grouping_consecutive_months where total_consecutive_months > 3;```

</details>


---

# 🔥 SQL Concepts Used

✅ CTE (WITH clause)  
✅ INNER JOIN  
✅ Window Functions  
✅ ROW_NUMBER()  
✅ DENSE_RANK()  
✅ DATE_FORMAT()  
✅ DATE_SUB()  
✅ INTERVAL  
✅ GROUP BY  
✅ Revenue Leakage Analysis


---

# 📌 Real World Use Case

This type of query is used in:

- SaaS billing systems
- Subscription platforms
- Telecom billing
- Insurance systems
- Cloud cost monitoring

Example:

A customer contract says:

```
Monthly Charge = ₹20,000
```

But billing system generated:

```
Jan → ₹15,000
Feb → ₹12,000
Mar → ₹10,000
```

Company lost:

```
₹23,000 revenue leakage
```

This query automatically detects such cases.

