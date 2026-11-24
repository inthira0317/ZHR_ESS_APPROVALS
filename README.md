# ZHR_ESS_APPROVALS
16-07-2026 -This function module is designed to retrieve the pending approval counts for an employee based on their approval role. It collects pending requests from multiple ESS processes, calculates individual process counts, and returns a consolidated approval summary.


Let's break `ZHR_ESS_APPROVALS` into individual coding blocks. I'll explain **what each block does, why it is required, how the ABAP statements work, and how the business logic flows**.

We will follow the original function module code rather than changing its business logic.

# **ZHR_ESS_APPROVALS – Coding explanation**

## **1. Function module declaration**

**Code:**

```
FUNCTION zhr_ess_approvals.
```

**Explanation:**

- `FUNCTION` is used to define a function module.
- `ZHR_ESS_APPROVALS` is the custom function module name.
- It is executed when another ABAP program, function module, or application calls it.

**Business purpose:** Provides a reusable interface to retrieve pending ESS approval counts.

## **2. Importing and table parameters**

**Code:**

```
*" Local Interface:
*" IMPORTING
*"   VALUE(IM_PERNR) TYPE PERSNO OPTIONAL
*"   VALUE(IM_ROLE)  TYPE CHAR100 OPTIONAL
*" TABLES
*"   EX_PENDING_LIST STRUCTURE ZHR_ESS_APPROVAL OPTIONAL
```

**Explanation:**

There are two input parameters and one output table parameter.

| Parameter         | Direction | Purpose                          |
| ----------------- | --------- | -------------------------------- |
| `IM_PERNR`        | Importing | Approver's personnel number      |
| `IM_ROLE`         | Importing | Approver's role                  |
| `EX_PENDING_LIST` | Tables    | Returns pending approval summary |

Example input:

```
IM_PERNR = '00001234'
IM_ROLE  = 'DIV'
```

The function module uses these inputs to identify the approver and determine which requests belong to them.

## **3. Determine approval status based on role**

**Code:**

```
IF im_role = 'DIV'.
  gv_status_id = '01'.

ELSEIF im_role = 'DIVE'.
  gv_status_id = '02'.
```

**Explanation:**

The function module checks the role passed by the calling application.

| Role | Meaning                     | Status |
| ---- | --------------------------- | ------ |
| DIV  | Division-level approver     | 01     |
| DIVE | Division executive approver | 02     |

**How it works:**

1. If the employee has role `DIV`, assign status `01`.
2. Otherwise, if the employee has role `DIVE`, assign status `02`.
3. The assigned status is used in subsequent database queries.

**Important:** The status is used to identify the approval stage, not necessarily the final status of an application.

## **4. Retrieve Shift Change applications**

**Code:**

```
IF im_role = 'DIV'.

  gv_status_id = '01'.

ELSEIF im_role = 'DIVE'.

  gv_status_id = '02'.

  CLEAR: gv_len.

  SELECT *
    FROM zhr_shift_change
    INTO TABLE gt_shift_change
    WHERE zstatus_id = '01'
      AND zapprover = im_pernr.
```

**Explanation:**

This query is executed only when the role is `DIVE`.

The `SELECT` statement retrieves records from `ZHR_SHIFT_CHANGE`.

Conditions:

- `ZSTATUS_ID = '01'`: Only requests with status 01.
- `ZAPPROVER = IM_PERNR`: Only requests assigned to the current approver.

**Understanding the statements:**

| Statement      | Purpose                                 |
| -------------- | --------------------------------------- |
| `CLEAR gv_len` | Resets the previous count               |
| `SELECT *`     | Retrieves all matching database records |
| `INTO TABLE`   | Stores records in an internal table     |
| `WHERE`        | Filters the database records            |

### **4.1 Count the Shift Change records**

**Code:**

```
DESCRIBE TABLE gt_shift_change LINES gv_len.
```

This statement determines how many records are present in the internal table.

Example:

```
GT_SHIFT_CHANGE contains 3 records.

GV_LEN = 3
```

### **4.2 Prepare the output record**

**Code:**

```
gs_ess_approval-zcount = gv_len.

gs_ess_approval-zprocess_name = 'Shift Change'.

APPEND gs_ess_approval TO gt_ess_approval.
```

**Explanation:**

- Assign the number of requests to `ZCOUNT`.
- Assign the process name to `ZPROCESS_NAME`.
- Append the work area to the output internal table.

### **4.3 Update the total count**

**Code:**

```
gv_tot = gv_tot + gv_len.

CLEAR: gs_ess_approval, gv_len.
```

Suppose Shift Change has three pending applications.

```
GV_TOT = 0 + 3
GV_TOT = 3
```

The `CLEAR` statement resets the work area and count variable before processing the next category.

## **5. Retrieve WeekOff Change applications**

**Code:**

```
SELECT *
  FROM zhr_woff_change
  INTO TABLE gt_woff_change
  WHERE zstatus_id = '01'
    AND zapprover = im_pernr.
```

**Explanation:**

This query retrieves WeekOff Change requests from `ZHR_WOFF_CHANGE`.

Conditions:

- Status ID must be `01`.
- Approver must match the personnel number passed to the function module.

### **5.1 Calculate the count**

```
DESCRIBE TABLE gt_woff_change LINES gv_len.
```

### **5.2 Prepare the output**

```
gs_ess_approval-zcount = gv_len.

gs_ess_approval-zprocess_name = 'WeekOff Change'.

APPEND gs_ess_approval TO gt_ess_approval.
```

### **5.3 Update the total**

```
gv_tot = gv_tot + gv_len.

CLEAR: gs_ess_approval, gv_len.
```

**Example:**

```
Shift Change = 3
WeekOff Change = 2

GV_TOT = 3 + 2
GV_TOT = 5
```

Both Shift Change and WeekOff Change are processed only for DIVE.

## **6. Retrieve Leave Application requests**

**Code:**

```
SELECT *
  FROM zhr_absences_r
  INTO TABLE gt_absences_r
  WHERE zstatus_id = gv_status_id
    AND zapprover = im_pernr
    AND zinfotype NE '0015'
    AND subty NOT IN ('3700','3600','3500').
```

**Explanation:**

This query retrieves leave applications from `ZHR_ABSENCES_R`.

Let's understand each condition.

| Condition                   | Explanation                                 |
| --------------------------- | ------------------------------------------- |
| `ZSTATUS_ID = GV_STATUS_ID` | Matches the approval stage                  |
| `ZAPPROVER = IM_PERNR`      | Retrieves requests for the current approver |
| `ZINFOTYPE NE '0015'`       | Excludes Infotype 0015                      |
| `SUBTY NOT IN (...)`        | Excludes the three specified subtypes       |

`NE` means Not Equal.

`NOT IN` excludes all the listed values.

### **6.1 Calculate the leave count**

```
DESCRIBE TABLE gt_absences_r LINES gv_len.
```

### **6.2 Prepare the output record**

```
gs_ess_approval-zcount = gv_len.

gv_tot = gv_tot + gv_len.

gs_ess_approval-zprocess_name = 'Leave Application'.

gs_ess_approval-wfid = '010'.

APPEND gs_ess_approval TO gt_ess_approval.

CLEAR: gs_ess_approval, gv_len.
```

**Explanation:**

The function module:

1. Calculates the number of leave applications.
2. Adds the count to the overall total.
3. Assigns the process name.
4. Assigns workflow ID `010`.
5. Appends the result to the output table.

## **7. Retrieve COFF Application requests**

**Code:**

```
SELECT *
  FROM zhr_absences_r
  INTO TABLE gt_absences_r
  WHERE zstatus_id = gv_status_id
    AND zapprover = im_pernr
    AND zinfotype NE '0015'
    AND subty IN ('3700','3600','3500').
```

**Explanation:**

This query uses the same database table as Leave Application.

However, the subtype condition is different.

Previously:

```
subty NOT IN ('3700','3600','3500')
```

Now:

```
subty IN ('3700','3600','3500')
```

This means only the three specified subtypes are retrieved.

### **7.1 Calculate the COFF count**

```
DESCRIBE TABLE gt_absences_r LINES gv_len.
```

### **7.2 Prepare the output record**

```
gs_ess_approval-zcount = gv_len.

gs_ess_approval-wfid = '014'.

gv_tot = gv_tot + gv_len.

gs_ess_approval-zprocess_name = 'COFF Application'.

APPEND gs_ess_approval TO gt_ess_approval.

CLEAR: gs_ess_approval, gv_len.
```

**Business purpose:** Separates COFF applications from general leave applications using subtype classification.

## **8. Retrieve PH Wage Claim requests**

**Code:**

```
SELECT *
  FROM zhr_absences_r
  INTO TABLE gt_absences_r
  WHERE zstatus_id = gv_status_id
    AND zapprover = im_pernr
    AND zinfotype = '0015'.
```

**Explanation:**

This query identifies wage claim requests using Infotype `0015`.

Conditions:

- Approval status must match the role.
- Approver must match the personnel number.
- Infotype must be `0015`.

### **8.1 Calculate the count**

```
DESCRIBE TABLE gt_absences_r LINES gv_len.
```

### **8.2 Prepare the output record**

```
gs_ess_approval-zcount = gv_len.

gs_ess_approval-wfid = '013'.

gv_tot = gv_tot + gv_len.

gs_ess_approval-zprocess_name = 'PH Wage Claim'.

APPEND gs_ess_approval TO gt_ess_approval.

CLEAR: gs_ess_approval, gv_len.
```

**Business purpose:** Identifies and counts pending PH Wage Claim requests separately from other absence-related applications.

## **9. Retrieve OD Application requests**

This process has different selection conditions depending on the approver's role.

### **9.1 For DIV role**

**Code:**

```
IF im_role = 'DIV'.

  SELECT *
    FROM zhr_attds_r
    INTO TABLE gt_attds
    WHERE status_id = gv_status_id
      AND reporting_officer = im_pernr.
```

**Explanation:**

For DIV, the system checks the `REPORTING_OFFICER` field.

Only records assigned to the current reporting officer are retrieved.

### **9.2 For DIVE role**

**Code:**

```
ELSEIF im_role = 'DIVE'.

  SELECT *
    FROM zhr_attds_r
    INTO TABLE gt_attds
    WHERE status_id = gv_status_id
      AND sanctioning_officer = im_pernr.

ENDIF.
```

**Explanation:**

For DIVE, the system checks the `SANCTIONING_OFFICER` field instead.

This distinguishes the approval responsibility between the two roles.

### **9.3 Calculate the OD count**

```
DESCRIBE TABLE gt_attds LINES gv_len.
```

### **9.4 Prepare the output record**

```
gs_ess_approval-zcount = gv_len.

gs_ess_approval-wfid = '012'.

gv_tot = gv_tot + gv_len.

gs_ess_approval-zprocess_name = 'OD Application'.

APPEND gs_ess_approval TO gt_ess_approval.

CLEAR: gs_ess_approval, gv_len.
```

**Business purpose:** Retrieves OD applications based on the employee's reporting or sanctioning authority.

## **10. Calculate Total Application Count**

**Code:**

```
gs_ess_approval-zcount = gv_tot.

gs_ess_approval-zprocess_name = 'Total Application'.

APPEND gs_ess_approval TO gt_ess_approval.
```

**Explanation:**

By this stage, all individual process counts have been accumulated in `GV_TOT`.

For example:

```
Shift Change       = 3
WeekOff Change     = 2
Leave Application  = 8
COFF Application   = 4
PH Wage Claim      = 5
OD Application     = 6
--------------------------------
Total              = 28
```

The total is assigned to `ZCOUNT` and appended as the final record.

## **11. Return the output table**

**Code:**

```
ex_pending_list[] = gt_ess_approval[].
```

**Explanation:**

This statement copies the complete output internal table into the function module's TABLES parameter.

The calling application receives the process-wise approval counts and total count.

# **12. Complete execution flow**

**CALL FUNCTION: ZHR_ESS_APPROVALS**

Input: IM_PERNR, IM_ROLE

**Determine approval role**

DIV → 01 | DIVE → 02

**DIVE-only processes**

**Shift Change**

**WeekOff Change**

**Common approval processes**

Leave Application

COFF Application

PH Wage Claim

OD Application

**Calculate GV_TOT**

Accumulate individual process counts

**Return EX_PENDING_LIST**

Process-wise counts + Total Application



Next, we can take each coding block and learn how to implement the same functionality using modern ABAP syntax (`VALUE`, `COND`, inline declarations, `LINES`, and reusable methods).
