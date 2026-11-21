## 6. Data Dictionary (Logical Model)

| Entity | Column Name | Key Type | Description / Meaning | Required? | Unique? |
| :--- | :--- | :---: | :--- | :---: | :---: |
| `DEPARTMENTS` | `department_id` | PK | Unique identifier for an organisational department | Yes | Yes |
| `DEPARTMENTS` | `department_name` | - | Name of the department | Yes | Yes |
| `DEPARTMENTS` | `office_code` | - | Physical office building and room number | Yes | No |
| `EMPLOYEES` | `employee_id` | PK | Unique system identifier for an employee | Yes | Yes |
| `EMPLOYEES` | `username` | UK | Corporate network login identity | Yes | Yes |
| `EMPLOYEES` | `department_id` | FK | Link to employee's assigned department | Yes | No |
| `EMPLOYEES` | `device_id` | - | Hardware identifier of registered corporate computer | Yes | No |
| `EMPLOYEES` | `account_status` | - | Operational state (ACTIVE, LOCKED, SUSPENDED) | Yes | No |
| `AUTH_LOGS` | `log_id` | PK | Unique sequential log entry identifier | Yes | Yes |
| `AUTH_LOGS` | `employee_id` | FK | Link to employee attempting authentication | Yes | No |
| `AUTH_LOGS` | `login_timestamp` | - | Date and time the authentication attempt occurred | Yes | No |
| `AUTH_LOGS` | `ip_address` | - | Network IP address of origin | Yes | No |
| `AUTH_LOGS` | `country` | - | Geolocation country code of origin | Yes | No |
| `AUTH_LOGS` | `success_flag` | - | Authentication result (1 = Success, 0 = Failure) | Yes | No |
| `SECURITY_ALERTS` | `alert_id` | PK | Unique alert record identifier | Yes | Yes |
| `SECURITY_ALERTS` | `employee_id` | FK | Link to target employee profile | Yes | No |
| `SECURITY_ALERTS` | `log_id` | FK | Link to triggering authentication event log | Yes | No |
| `SECURITY_ALERTS` | `alert_type` | - | Threat classification (REPEATED_FAILED_LOGINS, AFTER_HOURS) | Yes | No |
| `SECURITY_ALERTS` | `severity` | - | Impact rating (LOW, MEDIUM, HIGH, CRITICAL) | Yes | No |
| `SECURITY_ALERTS` | `detected_at` | - | Timestamp when the security engine generated the alert | Yes | No |
| `QUARANTINED_ACCOUNTS` | `quarantine_id` | PK | Unique account lock audit record | Yes | Yes |
| `QUARANTINED_ACCOUNTS` | `employee_id` | FK | Link to locked employee profile | Yes | No |
| `QUARANTINED_ACCOUNTS` | `reason` | - | Automated trigger cause for account quarantine | Yes | No |
| `QUARANTINED_ACCOUNTS` | `locked_at` | - | Timestamp when the account was locked by PL/SQL | Yes | No |
