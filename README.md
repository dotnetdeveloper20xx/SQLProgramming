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

# Real-World SQL Project Scenarios (Projects 1–8 with Full Code)

This master document includes eight full SQL projects. Each project provides:
- 🎯 Project Goal
- 🧩 Database Schema (CREATE TABLE scripts)
- 🔧 Feature-wise SQL Commands
- ⚙️ Stored Procedures, Triggers, Advanced Queries
- 📊 Complex Reporting Scenarios

---


## 🔷 Project 1: Human Resources Analytics System

### 🎯 Goal:
Automate HR analytics including salary tracking, department mapping, and employee leave insights.

### 🧩 Schema
```sql
CREATE TABLE Departments (
  DeptID INT PRIMARY KEY,
  DeptName VARCHAR(100)
);

CREATE TABLE Employees (
  EmployeeID INT PRIMARY KEY,
  FirstName VARCHAR(50),
  LastName VARCHAR(50),
  Salary DECIMAL(10,2),
  HireDate DATE,
  ManagerID INT,
  DepartmentID INT FOREIGN KEY REFERENCES Departments(DeptID)
);

CREATE TABLE SalaryAudit (
  AuditID INT IDENTITY PRIMARY KEY,
  EmployeeID INT,
  OldSalary DECIMAL(10,2),
  NewSalary DECIMAL(10,2),
  ChangedAt DATETIME DEFAULT GETDATE()
);

CREATE TABLE LeaveRequests (
  RequestID INT PRIMARY KEY,
  EmployeeID INT FOREIGN KEY REFERENCES Employees(EmployeeID),
  LeaveDate DATE,
  Status VARCHAR(20)
);
```

### 🔧 Trigger: Salary Change Audit
```sql
CREATE TRIGGER trg_SalaryAudit
ON Employees
AFTER UPDATE
AS
BEGIN
  INSERT INTO SalaryAudit (EmployeeID, OldSalary, NewSalary)
  SELECT d.EmployeeID, d.Salary, i.Salary
  FROM DELETED d
  JOIN INSERTED i ON d.EmployeeID = i.EmployeeID
  WHERE d.Salary <> i.Salary;
END;
```

### 📊 Report: Top 3 Earners per Department
```sql
WITH RankedSalaries AS (
  SELECT *, RANK() OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS rnk
  FROM Employees
)
SELECT FirstName, LastName, Salary, DepartmentID FROM RankedSalaries WHERE rnk <= 3;
```

### 🔁 Recursive CTE: Leave Streak
```sql
WITH RecLeaves AS (
  SELECT EmployeeID, LeaveDate, 1 AS LeaveStreak
  FROM LeaveRequests
  WHERE Status = 'Approved'

  UNION ALL

  SELECT L.EmployeeID, L.LeaveDate, R.LeaveStreak + 1
  FROM LeaveRequests L
  JOIN RecLeaves R ON L.EmployeeID = R.EmployeeID AND L.LeaveDate = DATEADD(DAY, 1, R.LeaveDate)
)
SELECT * FROM RecLeaves;
```

---

## 🟩 Project 2: E-Commerce Analytics Dashboard

### 🎯 Goal:
Provide insights into customer spend, product performance, and invoice generation.

### 🧩 Schema
```sql
CREATE TABLE Customers (
  CustomerID INT PRIMARY KEY,
  Name VARCHAR(100),
  Segment VARCHAR(50),
  CreatedDate DATE
);

CREATE TABLE Products (
  ProductID INT PRIMARY KEY,
  Name VARCHAR(100),
  Price DECIMAL(10,2),
  CategoryID INT
);

CREATE TABLE Orders (
  OrderID INT PRIMARY KEY,
  CustomerID INT FOREIGN KEY REFERENCES Customers(CustomerID),
  OrderDate DATE,
  TotalAmount DECIMAL(10,2)
);

CREATE TABLE OrderItems (
  OrderID INT FOREIGN KEY REFERENCES Orders(OrderID),
  ProductID INT FOREIGN KEY REFERENCES Products(ProductID),
  Quantity INT,
  Price DECIMAL(10,2)
);
```

### 🔧 JSON: Customer Order Report
```sql
SELECT CustomerID, Name,
  (SELECT OrderID, OrderDate, TotalAmount
   FROM Orders O WHERE O.CustomerID = C.CustomerID
   FOR JSON PATH) AS OrdersJSON
FROM Customers C;
```

### 📊 Monthly Product Leaderboard
```sql
WITH MonthlyRevenue AS (
  SELECT ProductID, FORMAT(OrderDate, 'yyyy-MM') AS SaleMonth, SUM(Quantity * Price) AS Revenue
  FROM Orders O JOIN OrderItems OI ON O.OrderID = OI.OrderID
  GROUP BY ProductID, FORMAT(OrderDate, 'yyyy-MM')
),
Ranked AS (
  SELECT *, RANK() OVER (PARTITION BY SaleMonth ORDER BY Revenue DESC) AS Rank
  FROM MonthlyRevenue
)
SELECT * FROM Ranked WHERE Rank = 1;
```

### ⚙️ Trigger: Prevent Product Deletion
```sql
CREATE TRIGGER trg_ProtectProduct
ON Products
INSTEAD OF DELETE
AS
BEGIN
  IF EXISTS (
    SELECT 1 FROM OrderItems WHERE ProductID IN (SELECT ProductID FROM DELETED)
  )
    RAISERROR('Cannot delete product with existing sales.', 16, 1);
  ELSE
    DELETE FROM Products WHERE ProductID IN (SELECT ProductID FROM DELETED);
END;
```

---





## 🟥 Project 3: Compliance & Audit Tracking System

### 🎯 Goal:
Secure access logs and monitor user activities.

### 🧩 Schema
```sql
CREATE TABLE Users (
  UserID INT PRIMARY KEY,
  Username VARCHAR(50),
  Role VARCHAR(50)
);

CREATE TABLE UserLogins (
  LoginID INT PRIMARY KEY,
  UserID INT FOREIGN KEY REFERENCES Users(UserID),
  LoginTime DATETIME
);

CREATE TABLE ChangeLog (
  LogID INT IDENTITY PRIMARY KEY,
  TableName VARCHAR(100),
  ChangeType VARCHAR(50),
  ChangedAt DATETIME DEFAULT GETDATE()
);

CREATE TABLE Products (
  ProductID INT PRIMARY KEY,
  Name VARCHAR(100),
  Price DECIMAL(10,2),
  ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
  ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON);
```

### 🔧 Features
```sql
-- Track logins in the past month
SELECT UserID, COUNT(*) AS LoginCount
FROM UserLogins
WHERE LoginTime > DATEADD(DAY, -30, GETDATE())
GROUP BY UserID;

-- Recover deleted data using SYSTEM_VERSIONING
SELECT * FROM Products FOR SYSTEM_TIME ALL WHERE ProductID = 101;

-- Log sensitive queries
DECLARE @sql NVARCHAR(MAX) = 'SELECT * FROM Users WHERE Role = ''Admin''';
INSERT INTO ChangeLog (TableName, ChangeType) VALUES ('Users', 'Sensitive Query');
EXEC sp_executesql @sql;
```

---

## 🟦 Project 4: Inventory & Warehouse Management

### 🎯 Goal:
Manage stock levels, movements, and restocking workflows.

### 🧩 Schema
```sql
CREATE TABLE Inventory (
  ProductID INT PRIMARY KEY,
  QuantityInStock INT,
  ReorderLevel INT
);

CREATE TABLE Suppliers (
  SupplierID INT PRIMARY KEY,
  Name VARCHAR(100),
  Country VARCHAR(50)
);

CREATE TABLE RestockEvents (
  EventID INT PRIMARY KEY,
  ProductID INT,
  SupplierID INT,
  QuantityAdded INT,
  RestockDate DATE
);

CREATE TABLE InventoryMovements (
  MovementID INT PRIMARY KEY,
  ProductID INT,
  MovementType VARCHAR(10),
  Quantity INT,
  MovementDate DATE
);
```

### 🔧 Features
```sql
-- Products below reorder level
SELECT ProductID, QuantityInStock FROM Inventory WHERE QuantityInStock < ReorderLevel;

-- Total restock per supplier
SELECT SupplierID, SUM(QuantityAdded) AS Total FROM RestockEvents GROUP BY SupplierID;

-- Inventory movement breakdown
SELECT ProductID, MovementType, SUM(Quantity) AS Qty FROM InventoryMovements GROUP BY ProductID, MovementType;

-- Prevent oversell trigger
CREATE TRIGGER trg_PreventOversell
ON InventoryMovements
INSTEAD OF INSERT
AS
BEGIN
  IF EXISTS (
    SELECT 1 FROM INSERTED I JOIN Inventory Inv ON I.ProductID = Inv.ProductID
    WHERE I.MovementType = 'OUT' AND I.Quantity > Inv.QuantityInStock
  )
    RAISERROR('Insufficient inventory.', 16, 1);
  ELSE
    INSERT INTO InventoryMovements SELECT * FROM INSERTED;
END;
```

---

## 🏥 Project 5: Healthcare Records and Appointments

### 🎯 Goal:
Track patient visits, doctor performance, and treatment history.

### 🧩 Schema
```sql
CREATE TABLE Patients (
  PatientID INT PRIMARY KEY,
  Name VARCHAR(100),
  DOB DATE,
  Gender CHAR(1)
);

CREATE TABLE Doctors (
  DoctorID INT PRIMARY KEY,
  Name VARCHAR(100),
  Specialty VARCHAR(100)
);

CREATE TABLE Appointments (
  AppointmentID INT PRIMARY KEY,
  PatientID INT,
  DoctorID INT,
  VisitDate DATE,
  Diagnosis VARCHAR(255)
);

CREATE TABLE Prescriptions (
  PrescriptionID INT PRIMARY KEY,
  AppointmentID INT,
  Medication VARCHAR(100),
  Dosage VARCHAR(50)
);
```

### 🔧 Features
```sql
-- Doctor monthly visit average
SELECT DoctorID, COUNT(*) / 12.0 AS AvgMonthlyVisits
FROM Appointments
GROUP BY DoctorID;

-- Patient record in JSON
SELECT P.PatientID, P.Name,
  (SELECT A.VisitDate, A.Diagnosis, PR.Medication, PR.Dosage
   FROM Appointments A
   JOIN Prescriptions PR ON A.AppointmentID = PR.AppointmentID
   WHERE A.PatientID = P.PatientID
   FOR JSON PATH) AS MedicalHistory
FROM Patients P;
```

---

## 🏦 Project 6: Banking Transactions & Fraud Detection

### 🎯 Goal:
Track financial transactions and detect suspicious behavior.

### 🧩 Schema
```sql
CREATE TABLE Accounts (
  AccountID INT PRIMARY KEY,
  CustomerID INT,
  Balance DECIMAL(12,2)
);

CREATE TABLE Transactions (
  TransactionID INT PRIMARY KEY,
  AccountID INT,
  Type VARCHAR(10),
  Amount DECIMAL(10,2),
  TransactionDate DATE
);
```

### 🔧 Features
```sql
-- Running balance
SELECT AccountID, TransactionDate, SUM(Amount) OVER (PARTITION BY AccountID ORDER BY TransactionDate) AS RunningBalance
FROM Transactions;

-- Flag large withdrawals
CREATE TRIGGER trg_LargeWithdrawal
ON Transactions
AFTER INSERT
AS
BEGIN
  IF EXISTS (
    SELECT 1 FROM INSERTED WHERE Type = 'Withdrawal' AND Amount > 10000
  )
    INSERT INTO ChangeLog (TableName, ChangeType) VALUES ('Transactions', 'HighValue Withdrawal');
END;
```

---

## 💳 Project 7: Subscription Billing Platform

### 🎯 Goal:
Manage invoices, payments, and subscription lifecycle.

### 🧩 Schema
```sql
CREATE TABLE Users (
  UserID INT PRIMARY KEY,
  Name VARCHAR(100),
  PlanID INT,
  SignupDate DATE,
  IsActive BIT
);

CREATE TABLE Plans (
  PlanID INT PRIMARY KEY,
  PlanName VARCHAR(50),
  MonthlyFee DECIMAL(8,2)
);

CREATE TABLE Invoices (
  InvoiceID INT PRIMARY KEY,
  UserID INT,
  InvoiceDate DATE,
  Amount DECIMAL(8,2)
);

CREATE TABLE Payments (
  PaymentID INT PRIMARY KEY,
  InvoiceID INT,
  PaymentDate DATE,
  PaymentAmount DECIMAL(8,2)
);
```

### 🔧 Features
```sql
-- Monthly recurring revenue
SELECT SUM(P.MonthlyFee) AS MRR
FROM Users U
JOIN Plans P ON U.PlanID = P.PlanID
WHERE U.IsActive = 1;

-- JSON report of all invoices and payments
SELECT U.UserID, U.Name,
  (SELECT InvoiceDate, Amount,
          (SELECT PaymentDate, PaymentAmount FROM Payments WHERE InvoiceID = I.InvoiceID FOR JSON PATH)
   FROM Invoices I WHERE I.UserID = U.UserID FOR JSON PATH) AS BillingHistory
FROM Users U;
```

---

## 🧮 Project 8: Loan Origination & Credit Scoring

### 🎯 Goal:
Evaluate applicant creditworthiness and track loan repayment.

### 🧩 Schema
```sql
CREATE TABLE Applicants (
  ApplicantID INT PRIMARY KEY,
  Name VARCHAR(100),
  DOB DATE,
  CreditScore INT
);

CREATE TABLE Loans (
  LoanID INT PRIMARY KEY,
  ApplicantID INT,
  LoanAmount DECIMAL(12,2),
  InterestRate DECIMAL(5,2),
  StartDate DATE,
  Status VARCHAR(20)
);

CREATE TABLE Payments (
  PaymentID INT PRIMARY KEY,
  LoanID INT,
  PaymentDate DATE,
  AmountPaid DECIMAL(10,2)
);
```

### 🔧 Features
```sql
-- Average credit score by age group
SELECT 
  CASE 
    WHEN DATEDIFF(YEAR, DOB, GETDATE()) < 30 THEN 'Under 30'
    WHEN DATEDIFF(YEAR, DOB, GETDATE()) BETWEEN 30 AND 50 THEN '30–50'
    ELSE '50+' END AS AgeGroup,
  AVG(CreditScore) AS AvgScore
FROM Applicants
GROUP BY 
  CASE 
    WHEN DATEDIFF(YEAR, DOB, GETDATE()) < 30 THEN 'Under 30'
    WHEN DATEDIFF(YEAR, DOB, GETDATE()) BETWEEN 30 AND 50 THEN '30–50'
    ELSE '50+' END;

-- Total paid and remaining loan balance
SELECT L.LoanID, L.LoanAmount, SUM(P.AmountPaid) AS TotalPaid,
       L.LoanAmount - SUM(P.AmountPaid) AS RemainingBalance
FROM Loans L
JOIN Payments P ON L.LoanID = P.LoanID
GROUP BY L.LoanID, L.LoanAmount;
```

---


