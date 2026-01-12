# Query Performance Optimization – Logical Improvements

## Overview

The original job search query suffered from slow execution due to excessive joins, row multiplication, and non-index-friendly keyword searches.  
This optimization focuses on **reducing unnecessary database work**, **preventing row explosion**, and **improving logical query flow** rather than micro-tuning SQL syntax.

---

## Problems in the Original Query

### 1. Row Explosion from Multiple `LEFT JOIN`s

- The query joined many one-to-many tables:
  - `jobs_personalities`
  - `jobs_practical_skills`
  - `jobs_basic_abilities`
  - `jobs_tools`
  - `jobs_career_paths`
  - qualification tables
- Each join multiplied rows before filtering
- This forced the use of `GROUP BY Jobs.id` to remove duplicates

**Impact**
- Large intermediate result sets  
- High memory usage  
- Poor performance as data grows  

---

### 2. `GROUP BY` Used Only for Deduplication

- `GROUP BY Jobs.id` was not used for aggregation
- It existed solely to counteract duplicated rows caused by joins

**Impact**
- Extra sorting
- Temporary tables
- Slower execution

---

### 3. Keyword Search Applied After All Joins

- Keyword filtering (`LIKE '%keyword%'`) happened **after** joining all related tables
- Most joined rows were unnecessary because only a small subset matched the keyword

**Impact**
- Database processed far more data than required

---

## Key Logical Improvements

---

## 1. Replace `LEFT JOIN + GROUP BY` with `EXISTS`

### Old Logic

> Join everything → filter → group to remove duplicates

### New Logic

> Check if related data exists **only when needed**

```sql
EXISTS (
  SELECT 1
  FROM jobs_personalities jp
  JOIN personalities p ON p.id = jp.personality_id
  WHERE jp.job_id = Jobs.id
    AND p.name LIKE '%keyword%'
)
```

**Benefits**
- No row multiplication
- No need for `GROUP BY`
- Subqueries stop execution on the first match

---

## 2. Join Only What Is Always Required

### Kept as `INNER JOIN`
- `job_categories`
- `job_types`

Reasons:
- One-to-one relationship
- Required for every job record
- Indexed foreign keys

All optional relationships were moved into `EXISTS` clauses.

---

## 3. Filter Early, Not Late

The optimized query:
- Applies `publish_status` and `deleted` filters immediately
- Narrows down candidate jobs before evaluating related tables

**Result**
- Fewer rows participate in expensive checks

---

## 4. Separate Search Logic from Data Expansion (Conceptually)

The optimized query follows this logical flow:

1. Identify **which jobs match** search criteria
2. Ensure each job appears **only once**
3. Avoid loading relationship data unless required for matching

This mirrors how scalable search systems are designed.

---

## 5. Index-Friendly Query Structure

The new query structure enables efficient index usage on:

- `jobs (publish_status, deleted, sort_order, id)`
- Foreign keys in join tables
- `affiliates (type, deleted, id)`

Even though `%keyword%` still limits index usage, the total scanned rows are greatly reduced.

---

## Summary

This optimization focuses on **logical restructuring**:

- Avoid joining data you only need to check
- Replace row multiplication with existence checks
- Remove `GROUP BY` used as a workaround
- Reduce total work done by the database

The result is a query that is **faster**, **cleaner**, and **scales predictably** as data grows.
