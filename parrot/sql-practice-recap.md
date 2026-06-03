# SQL Practice Recap — Azure Ops Interview Prep

## Database
```
sqlite3 /Users/aakashmac/DevOps/Github/parrot/azure_ops_practice.db
```
Shorthand used: `sql "QUERY"` = `sqlite3 azure_ops_practice.db -header -column "QUERY"`

## Tables
- `virtual_machines` (id, vm_name, resource_group, environment, os_type, size, region)
- `engineers` (id, name, team, shift)
- `incidents` (id, title, severity, assigned_to, created_date, resolved_date, category)
- `backup_jobs` (id, vm_name, status, backup_date, vault_name)
- `pipeline_runs` (id, pipeline_name, source_table, target_table, status, rows_read, rows_written, run_date, duration_seconds)

---

## 1. Basic SELECT & Exploration
```sql
SELECT id, vm_name, environment FROM virtual_machines;
```

## 2. GROUP BY + COUNT
```sql
-- Count VMs per environment
SELECT id, vm_name, environment, COUNT(*) AS count
FROM virtual_machines
WHERE environment LIKE '%Prod%'
GROUP BY environment;
```

## 3. GROUP BY + HAVING (filter after aggregation)
```sql
-- VMs with 2+ failed backups
SELECT vm_name, COUNT(status)
FROM backup_jobs
WHERE status = 'Failed'
GROUP BY vm_name
HAVING COUNT(status) >= 2;
```
**Rule**: WHERE filters rows BEFORE grouping. HAVING filters AFTER grouping.

## 4. LEFT JOIN + NULL Check (Anti-Join Pattern)
```sql
-- Production VMs with NO backup jobs at all
SELECT v.vm_name
FROM virtual_machines AS v
LEFT JOIN backup_jobs AS b ON v.vm_name = b.vm_name
WHERE b.vm_name IS NULL
  AND v.environment = 'Production';
```
**Pattern**: LEFT JOIN + WHERE right_table.column IS NULL = "find rows with no match"

## 5. Data Validation — Source vs Target Row Mismatch
```sql
-- Pipelines where rows_read != rows_written (data loss/issue)
SELECT pipeline_name, run_date
FROM pipeline_runs
WHERE rows_read <> rows_written;
```

## 6. Finding Duplicates
```sql
-- Find duplicate vm_names (if any)
SELECT vm_name
FROM virtual_machines
GROUP BY vm_name
HAVING COUNT(vm_name) > 1;
```

## 7. NULL Checks — Find Missing Data
```sql
-- Unresolved incidents
SELECT * FROM incidents WHERE resolved_date IS NULL;
```

## 8. Date Filtering + Aggregation
```sql
-- Incident count by severity (resolved in 2026)
SELECT severity, COUNT(*)
FROM incidents
WHERE resolved_date >= '2026-01-01'
GROUP BY severity;
```

## 9. Multi-Table JOIN
```sql
-- Incidents with engineer names
SELECT i.title, i.severity, e.name
FROM incidents AS i
LEFT JOIN engineers AS e ON i.assigned_to = e.id;
```

## 10. INNER JOIN vs LEFT JOIN — Understanding the Difference
```sql
-- INNER JOIN: only incidents that have an assigned engineer
SELECT i.title, e.name
FROM incidents AS i
JOIN engineers AS e ON e.id = i.assigned_to;

-- LEFT JOIN from engineers: shows ALL engineers, even those with no incidents
SELECT e.name, e.shift, i.title, i.severity
FROM engineers AS e
LEFT JOIN incidents AS i ON e.id = i.assigned_to;
```

## 11. Subquery with IN
```sql
-- Engineers assigned to P1 incidents
SELECT name, team
FROM engineers
WHERE id IN (SELECT assigned_to FROM incidents WHERE severity = 'P1');
```

## 12. CASE WHEN — Conditional Logic
```sql
-- Label each backup as Good/Critical
SELECT vm_name,
    CASE
        WHEN status = 'Success' THEN 'Good'
        ELSE 'Critical'
    END AS health_status
FROM backup_jobs;
```
**Syntax**: CASE WHEN ... THEN ... ELSE ... END

## 13. CASE WHEN + SUM — Pivot Aggregation (Important Pattern)
```sql
-- Success vs Failure count per VM
SELECT vm_name,
    SUM(CASE WHEN status = 'Success' THEN 1 ELSE 0 END) AS success_count,
    SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END) AS fail_count
FROM backup_jobs
GROUP BY vm_name
ORDER BY fail_count DESC;
```

## 14. Subquery + CASE WHEN — Backup Health Report
```sql
-- Classify each VM's backup health based on success vs failure ratio
SELECT vm_name, success_count, fail_count,
    CASE
        WHEN fail_count = 0 THEN 'Good'
        WHEN fail_count > success_count THEN 'Critical'
        ELSE 'At Risk'
    END AS health
FROM (
    SELECT vm_name,
        SUM(CASE WHEN status = 'Success' THEN 1 ELSE 0 END) AS success_count,
        SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END) AS fail_count
    FROM backup_jobs
    GROUP BY vm_name
) sub
ORDER BY fail_count DESC;
```
**Rule**: Column aliases from SELECT can't be reused in the same SELECT. Wrap in a subquery to reference them.

## 15. ROW_NUMBER — Get Latest Record Per Group (TODO: Practice this)
```sql
-- Most recent backup per VM
SELECT vm_name, status, backup_date
FROM (
    SELECT vm_name, status, backup_date,
        ROW_NUMBER() OVER (PARTITION BY vm_name ORDER BY backup_date DESC) AS rn
    FROM backup_jobs
) sub
WHERE rn = 1;
```
**Pattern**: PARTITION BY = "restart numbering for each group", ORDER BY = "how to rank within group"

---

## Key Rules to Remember

1. **SQL execution order**: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
2. **WHERE** filters rows before grouping. **HAVING** filters after grouping.
3. **GROUP BY** must include every non-aggregated column in SELECT.
4. **LEFT JOIN + WHERE IS NULL** = find records with no match (anti-join).
5. **Column aliases** can only be reused in ORDER BY / HAVING — not in SELECT or WHERE of the same query. Use a subquery to reference them.
6. **CASE ... WHEN ... THEN ... ELSE ... END** — END closes the CASE, goes inside SUM() if aggregating.
7. **JOINs connect tables via foreign keys** — always ask "which column in A references which column in B?"
8. **Duplicates**: GROUP BY + HAVING COUNT(*) > 1
9. **Data validation**: Compare row counts, check for NULLs, verify source vs target after ETL.
10. **ROW_NUMBER() OVER (PARTITION BY x ORDER BY y)** — for "latest/first per group" queries.

---

## Common Interview Query Scenarios (Azure Ops Context)

| Scenario | Pattern |
|----------|---------|
| "Which VMs have failing backups?" | GROUP BY + HAVING + WHERE status='Failed' |
| "Which Production VMs have no backup?" | LEFT JOIN + IS NULL |
| "Did the ETL pipeline lose data?" | WHERE rows_read <> rows_written |
| "Show me incident load per engineer" | JOIN + GROUP BY + COUNT |
| "Backup health report" | SUM + CASE WHEN + subquery |
| "Latest backup status per VM" | ROW_NUMBER + subquery |
| "Unresolved P1 incidents" | WHERE severity='P1' AND resolved_date IS NULL |
| "Engineers with no incidents assigned" | LEFT JOIN + IS NULL |
