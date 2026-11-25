# Debug: BENFT_PLN_ID C01772_0006 / C01772_0010 missing from `purse` (branch 1)

Run the steps in order. Each step adds one join or filter from branch 1 of `purse`.
The first step where the count drops to 0 (or the plan disappears) is the culprit.

Plans under test: `'C01772_0006','C01772_0010'`

---

## Step 0 – Does the plan exist in the plan core view? (source code + key)

```sql
SELECT BENFT_PLN_ID, BENFT_PLN_KEY, REC_SRC_CD
FROM BASE_DS.BASE_DS_BENFT_PLN_CORE_VU
WHERE BENFT_PLN_ID IN ('C01772_0006','C01772_0010');
```
- No rows -> plan missing from the view.
- `REC_SRC_CD` not in `('HR2','CONVEY','HRCATE')` -> Step 2 will drop it.

## Step 1 – Members in MFV joined to the plan view (no filters)

```sql
SELECT BEN_PLN.BENFT_PLN_ID, COUNT(*) AS cnt,
       MIN(MFV.EFCTV_DT) AS min_efctv, MAX(MFV.EXPIR_DT) AS max_expir
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
GROUP BY 1;
```
- 0 rows -> `BENFT_PLN_KEY` mismatch between MFV and the plan view.

## Step 2 – Add REC_SRC_CD filter

```sql
SELECT BEN_PLN.BENFT_PLN_ID, BEN_PLN.REC_SRC_CD, COUNT(*) AS cnt
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BEN_PLN.REC_SRC_CD IN ('HR2','CONVEY','HRCATE')
GROUP BY 1,2;
```

## Step 3 – Add member-level date filters

```sql
SELECT BEN_PLN.BENFT_PLN_ID, COUNT(*) AS cnt
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BEN_PLN.REC_SRC_CD IN ('HR2','CONVEY','HRCATE')
  AND YEAR(MFV.BENFT_PLN_EFCTV_EXPIR_DT) > 2023
  AND MFV.EFCTV_DT >= '1/1/2025'
  AND MFV.EFCTV_DT < MFV.EXPIR_DT
  AND MFV.EFCTV_DT <= CURRENT_DATE + 60
  AND MFV.EXPIR_DT > CURRENT_DATE
GROUP BY 1;
```

### Step 3a – Which individual member filter kills it? (one-shot breakdown)

```sql
SELECT BEN_PLN.BENFT_PLN_ID,
  COUNT(*)                                                           AS total,
  SUM(IFF(YEAR(MFV.BENFT_PLN_EFCTV_EXPIR_DT) > 2023,1,0))            AS pass_year,
  SUM(IFF(MFV.EFCTV_DT >= '1/1/2025',1,0))                           AS pass_efctv_2025,
  SUM(IFF(MFV.EFCTV_DT < MFV.EXPIR_DT,1,0))                          AS pass_efctv_lt_expir,
  SUM(IFF(MFV.EFCTV_DT <= CURRENT_DATE + 60,1,0))                    AS pass_efctv_le_today60,
  SUM(IFF(MFV.EXPIR_DT > CURRENT_DATE,1,0))                          AS pass_expir_gt_today
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BEN_PLN.REC_SRC_CD IN ('HR2','CONVEY','HRCATE')
GROUP BY 1;
```
The column with the lowest number is the filter doing the damage.

## Step 4 – Look at the BPEF rows themselves

```sql
SELECT BENFT_PLN_KEY, EFCTV_DT, EXPIR_DT, ACTV_INDC
FROM CIM.BENFT_PLN_EFCTV
WHERE BENFT_PLN_KEY IN (
    SELECT BENFT_PLN_KEY
    FROM BASE_DS.BASE_DS_BENFT_PLN_CORE_VU
    WHERE BENFT_PLN_ID IN ('C01772_0006','C01772_0010'))
ORDER BY BENFT_PLN_KEY, EFCTV_DT;
```
- No rows -> the INNER JOIN to BPEF drops everything.
- `ACTV_INDC <> 1` -> dropped by `BPEF.ACTV_INDC = 1`.
- No row covering today / 1/1/2025 / 1/1/2026 -> dropped by the join date condition (Step 5).

## Step 5 – Add the BPEF join (with its ON conditions)

```sql
SELECT BEN_PLN.BENFT_PLN_ID, COUNT(*) AS cnt
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
JOIN CIM.BENFT_PLN_EFCTV BPEF
  ON BPEF.BENFT_PLN_KEY = MFV.BENFT_PLN_KEY
 AND (to_varchar('1/1/2025') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT
      OR CURRENT_DATE        BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT
      OR to_varchar('1/1/2026') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT)
 AND BPEF.ACTV_INDC = 1
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BEN_PLN.REC_SRC_CD IN ('HR2','CONVEY','HRCATE')
  AND YEAR(MFV.BENFT_PLN_EFCTV_EXPIR_DT) > 2023
  AND MFV.EFCTV_DT >= '1/1/2025'
  AND MFV.EFCTV_DT < MFV.EXPIR_DT
  AND MFV.EFCTV_DT <= CURRENT_DATE + 60
  AND MFV.EXPIR_DT > CURRENT_DATE
GROUP BY 1;
```

### Step 5a – Which BPEF join condition fails? (per BPEF row)

```sql
SELECT BEN_PLN.BENFT_PLN_ID, BPEF.EFCTV_DT, BPEF.EXPIR_DT, BPEF.ACTV_INDC,
  IFF(to_varchar('1/1/2025') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT,1,0)    AS covers_1_1_2025,
  IFF(CURRENT_DATE           BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT,1,0)    AS covers_today,
  IFF(to_varchar('1/1/2026') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT,1,0)    AS covers_1_1_2026
FROM BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
JOIN CIM.BENFT_PLN_EFCTV BPEF ON BPEF.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
ORDER BY 1, BPEF.EFCTV_DT;
```

## Step 6 – Add the final date block (full branch 1 minus SSBCI)

```sql
SELECT BEN_PLN.BENFT_PLN_ID, COUNT(*) AS cnt
FROM DFDS.DFDS_MBR_FULL_VU MFV
INNER JOIN BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
        ON MFV.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
JOIN CIM.BENFT_PLN_EFCTV BPEF
  ON BPEF.BENFT_PLN_KEY = MFV.BENFT_PLN_KEY
 AND (to_varchar('1/1/2025') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT
      OR CURRENT_DATE        BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT
      OR to_varchar('1/1/2026') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT)
 AND BPEF.ACTV_INDC = 1
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BEN_PLN.REC_SRC_CD IN ('HR2','CONVEY','HRCATE')
  AND YEAR(MFV.BENFT_PLN_EFCTV_EXPIR_DT) > 2023
  AND MFV.EFCTV_DT >= '1/1/2025'
  AND MFV.EFCTV_DT < MFV.EXPIR_DT
  AND MFV.EFCTV_DT <= CURRENT_DATE + 60
  AND MFV.EXPIR_DT > CURRENT_DATE
  AND (
        (   (BPEF.EXPIR_DT >= ADD_MONTHS(TO_DATE(to_varchar(CURRENT_DATE,'MM/DD/YYYY'),'MM/DD/YYYY'),0)
             AND BPEF.EFCTV_DT <= to_varchar(CURRENT_DATE,'MM/DD/YYYY'))
         OR BPEF.EFCTV_DT = ADD_MONTHS(TO_DATE(to_varchar(CURRENT_DATE,'MM/DD/YYYY'),'MM/DD/YYYY'),1)
         OR BPEF.EFCTV_DT BETWEEN CURRENT_DATE AND (to_varchar(CURRENT_DATE+1*60,'MM/DD/YYYY')) )
        OR to_varchar(CURRENT_DATE,'MM/DD/YYYY') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT
      )
GROUP BY 1;
```

### Step 6a – Which part of the final date block fails? (per BPEF row)

```sql
SELECT BEN_PLN.BENFT_PLN_ID, BPEF.EFCTV_DT, BPEF.EXPIR_DT,
  IFF(BPEF.EXPIR_DT >= CURRENT_DATE AND BPEF.EFCTV_DT <= to_varchar(CURRENT_DATE,'MM/DD/YYYY'),1,0) AS a_covers_today,
  IFF(BPEF.EFCTV_DT = ADD_MONTHS(CURRENT_DATE,1),1,0)                                              AS b_starts_next_month,
  IFF(BPEF.EFCTV_DT BETWEEN CURRENT_DATE AND (to_varchar(CURRENT_DATE+60,'MM/DD/YYYY')),1,0)       AS c_starts_within_60d,
  IFF(to_varchar(CURRENT_DATE,'MM/DD/YYYY') BETWEEN BPEF.EFCTV_DT AND BPEF.EXPIR_DT,1,0)           AS d_today_between
FROM BASE_DS.BASE_DS_BENFT_PLN_CORE_VU BEN_PLN
JOIN CIM.BENFT_PLN_EFCTV BPEF ON BPEF.BENFT_PLN_KEY = BEN_PLN.BENFT_PLN_KEY
WHERE BEN_PLN.BENFT_PLN_ID IN ('C01772_0006','C01772_0010')
  AND BPEF.ACTV_INDC = 1
ORDER BY 1, BPEF.EFCTV_DT;
```
If all of a-d are 0 for every row, the final date block is the culprit.

## Step 7 – Compare against a plan that DOES appear

Re-run Step 3a and Step 5a, replacing the plan IDs with a plan that shows up in the
output (e.g. `'C01772_0001'`). Compare which flags differ.

## Optional – Check the SSBCI CTE doesn't multiply or drop rows

It is a `LEFT JOIN`, so it can't drop rows, but it can duplicate them. `SELECT DISTINCT` at the top hides that.

```sql
SELECT COUNT(*), COUNT(DISTINCT MBR_ID)
FROM (
  SELECT DISTINCT
    CASE WHEN MBR_ID LIKE '%|%' THEN SPLIT_PART(MBR_ID,'|',1) ELSE SUBSTR(MBR_ID,1,10) END AS MBR_ID
  FROM QSF20_CMIDB001.CMI.SSBCI_MEMBER_LIST
);
```

---

## Reading the results

| Step where plan disappears | Cause | Fix |
|---|---|---|
| 0 | Plan not in core view | Data issue upstream |
| 1 | `BENFT_PLN_KEY` mismatch MFV vs plan view | Check key versioning |
| 2 | `REC_SRC_CD` not in the list | Add the code to the `IN` list |
| 3 / 3a | Member date filters | Identify the failing column in 3a |
| 4 / 5 / 5a | BPEF missing, inactive, or no date coverage (stale `1/1/2026`) | Update the hardcoded date or `LEFT JOIN` |
| 6 / 6a | Final BPEF date block | Adjust window or logic |
