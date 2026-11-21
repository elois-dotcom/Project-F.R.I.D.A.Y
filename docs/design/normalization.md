# Normalisation: Cybersecurity SQL Lab

This document takes the raw authentication data from an unnormalised flat structure to Third Normal Form (3NF). It ends with the five target tables: `DEPARTMENTS`, `EMPLOYEES`, `AUTH_LOGS`, `SECURITY_ALERTS`, and `QUARANTINED_ACCOUNTS`.

## 1. Unnormalised Form (UNF)

All employee, office, authentication, alert, and quarantine data sits in one flat structure, as it arrives from the raw CSV files.

**`RAW_AUTH_EVENT`**

| Column | Example Value | Problem |
| :--- | :--- | :--- |
| `Username` | `jsmith` | Repeated on every login row |
| `Device_ID` | `D001` | Repeated on every login row |
| `Office_Dept` | `East-170 / Finance` | Composite value (two facts in one column) |
| `Account_Status` | `ACTIVE` | Repeated on every login row |
| `Login_Date` | `2026-11-10` | Split from time |
| `Login_Time` | `14:32:05` | Split from date |
| `Country` | `RW` | - |
| `IP_Address` | `192.168.1.14` | - |
| `Success_Status` | `TRUE` / `1` | Mixed text representations |
| `Alert_Type`, `Severity`, `Detected_At` | `REPEATED_FAILED_LOGINS`, `CRITICAL` | Repeating group (0..n alerts per employee) |
| `Quarantine_Reason`, `Locked_At` | `Critical risk score` | Repeating group (0..n locks per employee) |

**Anomalies in this form:**

* **Update anomaly:** changing an employee's department means editing every one of their login rows.
* **Insertion anomaly:** a new employee cannot be recorded until they have logged in at least once.
* **Deletion anomaly:** deleting an employee's only login row also deletes the employee's profile.
* **Redundancy:** 12 exact duplicate rows, plus 4 spellings of the same office (`'East-170'`, `'east 170'`, `'E170'`, `'East_170'`).

## 2. First Normal Form (1NF)

**Rule:** every column holds a single atomic value, there are no repeating groups, and every row is uniquely identifiable.

**Actions taken:**

1. Split the composite `Office_Dept` into `Office` and `Department`.
2. Combine `Login_Date` and `Login_Time` into one atomic `Login_Timestamp` (Oracle `TIMESTAMP`).
3. Convert `Success_Status` (`'TRUE'`, `'FALSE'`, `'1'`, `'0'`) to a numeric `Success_Flag` (1 or 0).
4. Standardise the four office spellings to one canonical code.
5. Remove the 12 exact duplicate rows.
6. Move the repeating groups (alerts and quarantine events) into their own rows so each cell holds one value.
7. Assign a primary key. At this stage the identifying key is the composite (`Username`, `Login_Timestamp`).

**Result (1NF):**

**`AUTH_EVENT_1NF`**: PK (`Username`, `Login_Timestamp`)

| Column | Notes |
| :--- | :--- |
| `Username` | Part of PK |
| `Login_Timestamp` | Part of PK |
| `Device_ID` | Employee attribute |
| `Office` | Atomic, standardised |
| `Department` | Atomic |
| `Account_Status` | Employee attribute |
| `Country` | Event attribute |
| `IP_Address` | Event attribute |
| `Success_Flag` | 1 or 0 |
| `Alert_Type`, `Severity`, `Detected_At` | Nullable, alert attributes |
| `Quarantine_Reason`, `Locked_At` | Nullable, quarantine attributes |

**Dependency removed:** composite attributes and repeating groups. Every column now holds one atomic value.

## 3. Second Normal Form (2NF)

**Rule:** the table is in 1NF, and every non-key attribute depends on the *whole* primary key, not just part of it.

**Partial dependencies found** (the PK is the composite `Username` + `Login_Timestamp`):

| Determinant (part of PK) | Dependent attributes |
| :--- | :--- |
| `Username` | `Device_ID`, `Department`, `Office`, `Account_Status` |

These facts describe the *employee*, not the individual login event. They depend only on `Username`.

**Actions taken:**

1. Create an `EMPLOYEES` table holding the static profile attributes, keyed by a surrogate `Employee_ID` (with `Username` kept as a unique key).
2. Keep only event-level attributes in `AUTH_LOGS`, keyed by a surrogate `Log_ID`, with `Employee_ID` as a foreign key.
3. Move alert attributes into `SECURITY_ALERTS`, which depends on the triggering log event.
4. Move lock history into `QUARANTINED_ACCOUNTS`, which depends on the employee.

**Result (2NF):**

| Table | Primary Key | Non-key attributes |
| :--- | :--- | :--- |
| `EMPLOYEES` | `employee_id` | `username`, `device_id`, `department`, `office`, `account_status` |
| `AUTH_LOGS` | `log_id` | `employee_id` (FK), `login_timestamp`, `ip_address`, `country`, `success_flag` |
| `SECURITY_ALERTS` | `alert_id` | `employee_id` (FK), `log_id` (FK), `alert_type`, `severity`, `detected_at` |
| `QUARANTINED_ACCOUNTS` | `quarantine_id` | `employee_id` (FK), `reason`, `locked_at` |

**Dependency removed:** partial dependencies. Static employee profile details are now separate from individual authentication events.

## 4. Third Normal Form (3NF)

**Rule:** the table is in 2NF, and no non-key attribute depends on another non-key attribute (no transitive dependencies).

**Transitive dependency found in `EMPLOYEES`:**

```
employee_id → department → office
```

`office` is determined by `department`, not directly by the employee. If a department moves to a different office, every employee row in that department would need updating.

**Actions taken:**

1. Create a `DEPARTMENTS` table holding `department_name` and `office_code`.
2. Replace `department` and `office` in `EMPLOYEES` with a foreign key `department_id`.

**Result (3NF), final schema:**

| Table | PK | FKs | Other columns |
| :--- | :--- | :--- | :--- |
| `DEPARTMENTS` | `department_id` | - | `department_name`, `office_code` |
| `EMPLOYEES` | `employee_id` | `department_id` → `DEPARTMENTS` | `username` (UK), `device_id`, `account_status` |
| `AUTH_LOGS` | `log_id` | `employee_id` → `EMPLOYEES` | `login_timestamp`, `ip_address`, `country`, `success_flag` |
| `SECURITY_ALERTS` | `alert_id` | `employee_id` → `EMPLOYEES`, `log_id` → `AUTH_LOGS` | `alert_type`, `severity`, `detected_at` |
| `QUARANTINED_ACCOUNTS` | `quarantine_id` | `employee_id` → `EMPLOYEES` | `reason`, `locked_at` |

**Dependency removed:** transitive dependencies. Office codes and department names now live in a separate `DEPARTMENTS` entity.

## 5. Relationships

| Parent | Child | Cardinality | Meaning |
| :--- | :--- | :---: | :--- |
| `DEPARTMENTS` | `EMPLOYEES` | 1 : N | A department has many employees |
| `EMPLOYEES` | `AUTH_LOGS` | 1 : N | An employee has many login attempts |
| `EMPLOYEES` | `SECURITY_ALERTS` | 1 : N | An employee can trigger many alerts |
| `AUTH_LOGS` | `SECURITY_ALERTS` | 1 : N | A log event can be the trigger for alerts |
| `EMPLOYEES` | `QUARANTINED_ACCOUNTS` | 1 : N | An employee can be locked many times over time |

## 6. Summary of Normal Forms

| Form | Problem fixed | Key change |
| :--- | :--- | :--- |
| **UNF** | - | One flat table, repeating groups, composite values |
| **1NF** | Composite attributes, repeating groups, duplicates | Atomic columns, `Login_Timestamp`, `Success_Flag`, standardised offices |
| **2NF** | Partial dependency on `Username` | `EMPLOYEES` separated from `AUTH_LOGS` |
| **3NF** | Transitive dependency `Employee → Department → Office` | New `DEPARTMENTS` table |

## 7. Design Note

`SECURITY_ALERTS.employee_id` can also be derived through `log_id` (`alert_id → log_id → employee_id`), which is technically a transitive dependency. It is kept as a deliberate design choice so alerts can be queried per employee without an extra join. The PL/SQL trigger that locks accounts also reads it directly. If strict 3NF is required, drop the column and reach the employee through `AUTH_LOGS`.
