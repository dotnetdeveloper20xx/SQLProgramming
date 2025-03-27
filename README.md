# SQLProgramming
List the comman commands a developer should know.

## Example 1: Top 2 Highest Paid Employees per Department with Years Worked

### ✅ What:
Find the top 2 earners in each department and show how long they’ve worked.

### ❓ Why:
Common in dashboards or salary review systems.

### ⚖️ Explanation:
- `RANK() OVER (PARTITION BY ...)` assigns a rank within each department.
- `DATEDIFF(YEAR, HireDate, GETDATE())` calculates years of service.
- `+` joins first and last name.

### 🔧 SQL:
```sql
WITH RankedEmployees AS (
  SELECT
    E.EmployeeID,
    E.FirstName + ' ' + E.LastName AS FullName,
    D.DeptName,
    E.Salary,
    DATEDIFF(YEAR, E.HireDate, GETDATE()) AS YearsWorked,
    RANK() OVER (PARTITION BY D.DeptName ORDER BY E.Salary DESC) AS SalaryRank
  FROM Employees E
  INNER JOIN Departments D ON E.DepartmentID = D.DeptID
)
SELECT * FROM RankedEmployees WHERE SalaryRank <= 2;
```

### ✨ Related Enhancements:
- Use `DENSE_RANK()` if salaries may tie.
- Filter by `YearsWorked > 5` for veterans only.

---

## Example 2: Detect Consecutive Absenteeism Using `LAG()`

### ✅ What:
Find employees absent two days in a row.

### ❓ Why:
Helpful for HR monitoring and wellness policies.

### ⚖️ Explanation:
- `LAG()` pulls the previous row's date.
- `DATEDIFF(DAY, PreviousDate, AttendanceDate) = 1` checks for day-to-day absence.

### 🔧 SQL:
```sql
WITH Absences AS (
  SELECT
    EmployeeID,
    AttendanceDate,
    LAG(AttendanceDate) OVER (PARTITION BY EmployeeID ORDER BY AttendanceDate) AS PreviousDate
  FROM Attendance
  WHERE Status = 'Absent'
)
SELECT *
FROM Absences
WHERE DATEDIFF(DAY, PreviousDate, AttendanceDate) = 1;
```

### ✨ Related Enhancements:
- Add `COUNT(*) OVER (...)` to count streaks.
- Use `LEAD()` to check next-day status too.

---

## Example 3: Sync Salary Table Using `MERGE`

### ✅ What:
Synchronize salary adjustments by updating existing and inserting new rows.

### ❓ Why:
Efficient bulk syncing of data, like from external HR systems.

### ⚖️ Explanation:
- `MERGE` matches records and performs INSERT/UPDATE.
- `Source` and `Target` represent external and current tables.

### 🔧 SQL:
```sql
MERGE INTO Employees AS Target
USING SalaryAdjustments AS Source
ON Target.EmployeeID = Source.EmployeeID
WHEN MATCHED THEN
  UPDATE SET Salary = Source.NewSalary
WHEN NOT MATCHED THEN
  INSERT (EmployeeID, FirstName, LastName, HireDate, Salary, DepartmentID)
  VALUES (Source.EmployeeID, Source.FirstName, Source.LastName, Source.HireDate, Source.NewSalary, Source.DepartmentID);
```

### ✨ Related Enhancements:
- Add `OUTPUT INSERTED.*` to track changes.
- Wrap in transaction block for rollback capability.

---

## Example 4: Pivot Monthly Sales Per Product

### ✅ What:
Transform row-wise monthly sales into a columnar report.

### ❓ Why:
Used for dashboards and exportable reports.

### ⚖️ Explanation:
- `PIVOT()` converts values into column headers.
- `SUM()` aggregates data for each column.

### 🔧 SQL:
```sql
SELECT * FROM (
  SELECT ProductName, MONTH(SaleDate) AS SaleMonth, Revenue
  FROM Sales
) AS Source
PIVOT (
  SUM(Revenue) FOR SaleMonth IN ([1], [2], [3], [4], [5], [6], [7], [8], [9], [10], [11], [12])
) AS PivotTable;
```

### ✨ Related Enhancements:
- Use `FORMAT(SaleDate, 'MMM')` for month names.
- Add `ROLLUP` for yearly totals.

---

## Example 5: Categorize Customers by Purchase Frequency

### ✅ What:
Label customers as Frequent, Occasional, or Rare based on orders.

### ❓ Why:
Useful for customer segmentation and targeting.

### ⚖️ Explanation:
- `CASE` for custom category labels.
- `COUNT()` for order volume.

### 🔧 SQL:
```sql
SELECT
  C.CustomerID,
  C.Name,
  COUNT(O.OrderID) AS OrdersThisYear,
  CASE
    WHEN COUNT(O.OrderID) >= 10 THEN 'Frequent'
    WHEN COUNT(O.OrderID) BETWEEN 3 AND 9 THEN 'Occasional'
    ELSE 'Rare'
  END AS PurchaseCategory
FROM Customers C
LEFT JOIN Orders O ON C.CustomerID = O.CustomerID
  AND YEAR(O.OrderDate) = YEAR(GETDATE())
GROUP BY C.CustomerID, C.Name;
```

## 🔁 Recursive CTEs

### ✅ What:
A CTE that references itself to handle hierarchical or recursive data.

### ❓ Why:
Used for tasks like org charts, folder trees, or bill of materials.

### ⚖️ Syntax:
```sql
WITH RecursiveCTE AS (
  SELECT ... -- anchor member
  UNION ALL
  SELECT ... FROM RecursiveCTE JOIN ... -- recursive member
)
SELECT * FROM RecursiveCTE;
```

### 🔧 Examples:
```sql
-- Example 1: Organization hierarchy
WITH EmployeeTree AS (
  SELECT EmployeeID, ManagerID, FirstName, 0 AS Level
  FROM Employees WHERE ManagerID IS NULL
  UNION ALL
  SELECT E.EmployeeID, E.ManagerID, E.FirstName, T.Level + 1
  FROM Employees E JOIN EmployeeTree T ON E.ManagerID = T.EmployeeID
)
SELECT * FROM EmployeeTree;

-- Example 2: Folder structure
WITH FolderTree AS (
  SELECT FolderID, FolderName, ParentID FROM Folders WHERE ParentID IS NULL
  UNION ALL
  SELECT F.FolderID, F.FolderName, F.ParentID
  FROM Folders F JOIN FolderTree FT ON F.ParentID = FT.FolderID
)
SELECT * FROM FolderTree;

-- Example 3: Numeric sequence
WITH Numbers AS (
  SELECT 1 AS Num
  UNION ALL
  SELECT Num + 1 FROM Numbers WHERE Num < 10
)
SELECT * FROM Numbers;
```

---

## ⚙️ Triggers

### ✅ What:
SQL procedures that fire automatically on data changes.

### ❓ Why:
Used for enforcing business rules, logging, auditing.

### ⚖️ Syntax:
```sql
CREATE TRIGGER TriggerName ON TableName
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
  -- logic here
END;
```

### 🔧 Examples:
```sql
-- Example 1: Log salary updates
CREATE TRIGGER trg_LogSalaryUpdate ON Employees
AFTER UPDATE AS
BEGIN
  INSERT INTO SalaryAudit(EmployeeID, OldSalary, NewSalary, ChangedAt)
  SELECT d.EmployeeID, d.Salary, i.Salary, GETDATE()
  FROM DELETED d JOIN INSERTED i ON d.EmployeeID = i.EmployeeID
  WHERE d.Salary <> i.Salary;
END;

-- Example 2: Prevent deletion of important rows
CREATE TRIGGER trg_PreventDelete ON Products
INSTEAD OF DELETE AS
BEGIN
  RAISERROR('Cannot delete products directly.', 16, 1);
END;

-- Example 3: Audit new customers
CREATE TRIGGER trg_NewCustomerAudit ON Customers
AFTER INSERT AS
BEGIN
  INSERT INTO CustomerAudit(CustomerID, AuditDate)
  SELECT CustomerID, GETDATE() FROM INSERTED;
END;
```

---

## 🧩 Dynamic SQL

### ✅ What:
Builds and executes SQL strings at runtime.

### ❓ Why:
Used when table names, column lists, or filters are dynamic.

### ⚖️ Syntax:
```sql
DECLARE @sql NVARCHAR(MAX);
SET @sql = 'SELECT ...';
EXEC sp_executesql @sql;
```

### 🔧 Examples:
```sql
-- Example 1: Dynamic pivot
DECLARE @cols NVARCHAR(MAX), @sql NVARCHAR(MAX);
SELECT @cols = STRING_AGG(QUOTENAME(Month(SaleDate)), ',') FROM Sales;
SET @sql = 'SELECT ProductName, ' + @cols + ' FROM (...) PIVOT (...)';
EXEC sp_executesql @sql;

-- Example 2: Table name from variable
DECLARE @table NVARCHAR(100) = 'Employees';
DECLARE @sql NVARCHAR(MAX) = 'SELECT * FROM ' + QUOTENAME(@table);
EXEC sp_executesql @sql;

-- Example 3: Search across columns
DECLARE @search NVARCHAR(50) = '%John%';
SET @sql = 'SELECT * FROM Customers WHERE FirstName LIKE @search OR LastName LIKE @search';
EXEC sp_executesql @sql, N'@search NVARCHAR(50)', @search;
```

---

## 📦 JSON/XML Handling

### ✅ What:
Store, retrieve, and query JSON or XML data in SQL Server.

### ❓ Why:
Supports semi-structured data from APIs, logs, and NoSQL-style storage.

### ⚖️ Syntax:
```sql
-- JSON
SELECT JSON_VALUE(column, '$.key')
-- XML
SELECT column.value('(/root/node)[1]', 'VARCHAR(100)')
```

### 🔧 Examples:
```sql
-- Example 1: Extract from JSON
SELECT OrderID, JSON_VALUE(OrderDetails, '$.ProductName') AS Product
FROM Orders;

-- Example 2: Query XML
SELECT CustomerData.value('(/Customer/Email)[1]', 'VARCHAR(100)') AS Email
FROM Customers;

-- Example 3: Convert table to JSON
SELECT * FROM Employees FOR JSON PATH;
```

---

## 📅 Temporal Tables

### ✅ What:
System-versioned tables that keep history automatically.

### ❓ Why:
Allow tracking changes over time and querying historical data.

### ⚖️ Syntax:
```sql
CREATE TABLE TableName (
  ..., 
  ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
  ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON);
```

### 🔧 Examples:
```sql
-- Example 1: Enable temporal table
CREATE TABLE Products (
  ProductID INT PRIMARY KEY,
  Name VARCHAR(100),
  Price DECIMAL(10,2),
  ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
  ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON);

-- Example 2: Query product price history
SELECT * FROM Products
FOR SYSTEM_TIME BETWEEN '2023-01-01' AND '2023-12-31'
WHERE ProductID = 1001;

-- Example 3: See current and past rows
SELECT * FROM Products FOR SYSTEM_TIME ALL;
```

---

## 🛡️ Security & Auditing Queries

### ✅ What:
Techniques to enforce data security and track user actions.

### ❓ Why:
Used in regulated environments and internal monitoring.

### ⚖️ Syntax:
Use built-in metadata, auditing triggers, and login history views.

### 🔧 Examples:
```sql
-- Example 1: Find login history
SELECT * FROM sys.dm_exec_sessions WHERE is_user_process = 1;

-- Example 2: Track data changes
CREATE TRIGGER trg_DataAudit ON Employees
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
  INSERT INTO ChangeLog (TableName, ChangeType, ChangeDate)
  SELECT 'Employees', 'MODIFIED', GETDATE();
END;

-- Example 3: See permission grants
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
```

---





