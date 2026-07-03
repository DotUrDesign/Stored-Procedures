# SQL Server Stored Procedures – Advanced Concepts

This guide covers advanced concepts of SQL Server Stored Procedures, including:

- Creating Stored Procedures
- Parameterized Stored Procedures
- Default Parameters
- Multiple SQL Statements
- Variables
- IF...ELSE Conditions
- TRY...CATCH Error Handling

---

# 1. Creating a Basic Stored Procedure

A Stored Procedure is a precompiled SQL program stored in the database. It allows SQL statements to be executed repeatedly without rewriting them.

## Syntax

```sql
CREATE PROCEDURE ProcedureName
AS
BEGIN
    -- SQL Statements
END
```

## Example

```sql
CREATE PROCEDURE GetSummaryReport
AS
BEGIN
    SELECT
        COUNT(*) AS total_customers,
        AVG(score) AS avg_score
    FROM customers
    WHERE country = 'India';
END;
```

## Execute

```sql
EXEC GetSummaryReport;
```

---

# 2. Parameterized Stored Procedures

Parameters make Stored Procedures dynamic by allowing input values during execution.

## Example

```sql
CREATE PROCEDURE GetSummaryReport
    @country NVARCHAR(50)
AS
BEGIN
    SELECT
        COUNT(*) AS total_customers,
        AVG(score) AS avg_score
    FROM customers
    WHERE country = @country;
END;
```

## Execute

```sql
EXEC GetSummaryReport @country = 'India';
```

## Advantages

- Dynamic execution
- Code reusability
- Avoids hardcoding values

---

# 3. Default Parameters

Default values are used whenever a parameter is not explicitly supplied.

## Example

```sql
CREATE PROCEDURE GetSummaryReport
    @country NVARCHAR(50) = 'USA'
AS
BEGIN
    SELECT
        COUNT(*) AS total_customers,
        AVG(score) AS avg_score
    FROM customers
    WHERE country = @country;
END;
```

## Execute

```sql
EXEC GetSummaryReport @country = 'India';
```

or

```sql
EXEC GetSummaryReport;
```

If no value is passed,

```sql
@country = 'USA'
```

is automatically used.

---

# 4. Multiple SQL Statements Inside a Stored Procedure

A Stored Procedure can execute multiple SQL statements and return multiple result sets.

## Example

```sql
CREATE PROCEDURE GetCustomerSummary
    @country NVARCHAR(50) = 'USA'
AS
BEGIN

    SELECT
        COUNT(*) AS total_customers,
        AVG(score) AS avg_score
    FROM customers
    WHERE country = @country;

    SELECT
        COUNT(OrderId) AS TotalOrders,
        SUM(sales) AS TotalSales
    FROM Sales.Orders o
    JOIN Sales.Customers c
        ON o.CustomerId = c.CustomerId
    WHERE c.Country = @country;

END;
```

## Execute

```sql
EXEC GetCustomerSummary @country = 'India';
```

This procedure returns

- Customer Summary
- Sales Summary

---

# 5. Variables Inside Stored Procedures

Variables are used to store intermediate values that can be reused later.

## Declaring Variables

```sql
DECLARE @total_customers INT;
DECLARE @avg_score FLOAT;
```

## Example

```sql
CREATE PROCEDURE GetCustomerSummary
    @country NVARCHAR(50) = 'USA'
AS
BEGIN

    DECLARE
        @total_customers INT,
        @avg_score FLOAT;

    SELECT
        @total_customers = COUNT(*),
        @avg_score = AVG(score)
    FROM customers
    WHERE country = @country;

    PRINT 'Total Customers from '
          + @country
          + ' : '
          + CAST(@total_customers AS NVARCHAR);

    PRINT 'Average Score from '
          + @country
          + ' : '
          + CAST(@avg_score AS NVARCHAR);

END;
```

---

# 6. IF...ELSE in Stored Procedures

Conditional statements allow Stored Procedures to make decisions.

One common use case is **Data Cleansing** before generating reports.

## Example

```sql
CREATE PROCEDURE GetCustomerSummary
    @country NVARCHAR(50) = 'USA'
AS
BEGIN

    DECLARE
        @total_customers INT,
        @avg_score FLOAT;

    IF EXISTS
    (
        SELECT 1
        FROM Sales.Customers
        WHERE score IS NULL
          AND country = @country
    )
    BEGIN

        PRINT 'Updating NULL scores to 0';

        UPDATE Sales.Customers
        SET score = 0
        WHERE score IS NULL
          AND country = @country;

    END

    ELSE

    BEGIN

        PRINT 'No NULL scores found';

    END;

    SELECT
        @total_customers = COUNT(*),
        @avg_score = AVG(score)
    FROM customers
    WHERE country = @country;

    PRINT 'Total Customers from '
          + @country
          + ' : '
          + CAST(@total_customers AS NVARCHAR);

    PRINT 'Average Score from '
          + @country
          + ' : '
          + CAST(@avg_score AS NVARCHAR);

END;
```

---

# Why Use IF...ELSE?

Suppose some customer scores are NULL.

Instead of generating an incorrect report,

the procedure first:

✔ Checks for NULL values

↓

✔ Cleans the data

↓

✔ Generates the report

This is a common Data Engineering pattern.

---

# IF EXISTS

Instead of

```sql
SELECT COUNT(*)
```

we often write

```sql
IF EXISTS
(
    SELECT 1
    FROM Sales.Customers
    WHERE score IS NULL
)
```

Why?

Because `EXISTS` stops as soon as it finds the first matching row, making it more efficient than counting all matching rows.

---

# 7. TRY...CATCH Block

Production Stored Procedures should always handle unexpected errors gracefully.

Instead of crashing,

they should:

- Catch the error
- Log useful information
- Exit gracefully

---

# Syntax

```sql
BEGIN TRY

    -- SQL Statements

END TRY

BEGIN CATCH

    -- Error Handling

END CATCH
```

---

# Example

```sql
CREATE PROCEDURE GetCustomerSummary
    @country NVARCHAR(50) = 'USA'
AS
BEGIN

BEGIN TRY

    DECLARE
        @total_customers INT,
        @avg_score FLOAT;

    IF EXISTS
    (
        SELECT 1
        FROM Sales.Customers
        WHERE score IS NULL
          AND country = @country
    )
    BEGIN

        UPDATE Sales.Customers
        SET score = 0
        WHERE score IS NULL
          AND country = @country;

    END;

    SELECT
        @total_customers = COUNT(*),
        @avg_score = AVG(score),
        1/0       -- Intentional Error
    FROM customers
    WHERE country = @country;

END TRY

BEGIN CATCH

    PRINT 'An Error Occurred';

    PRINT 'Error Message : '
          + ERROR_MESSAGE();

    PRINT 'Error Number : '
          + CAST(ERROR_NUMBER() AS NVARCHAR);

    PRINT 'Error Line : '
          + CAST(ERROR_LINE() AS NVARCHAR);

    PRINT 'Error Procedure : '
          + ERROR_PROCEDURE();

END CATCH;

END;
```

---

# Why Was `1/0` Added?

The instructor intentionally added

```sql
1/0
```

to generate a **Divide by Zero** exception.

Without TRY...CATCH,

SQL Server would immediately terminate the procedure.

With TRY...CATCH,

the procedure enters the `CATCH` block and provides meaningful debugging information.

---

# Built-in Error Functions

| Function | Description |
|----------|-------------|
| `ERROR_MESSAGE()` | Returns the complete error message |
| `ERROR_NUMBER()` | Returns SQL Server error number |
| `ERROR_LINE()` | Returns line number where error occurred |
| `ERROR_PROCEDURE()` | Returns Stored Procedure name that generated the error |

These functions are available only inside the `CATCH` block.

---

# Typical Execution Flow

```text
Procedure Starts
        │
        ▼
Input Parameters
        │
        ▼
Data Cleansing (IF EXISTS)
        │
        ▼
Business Logic
        │
        ▼
Report Generation
        │
        ▼
Any Error?
      │      │
      │      ▼
      │   CATCH Block
      │
      ▼
Procedure Ends Successfully
```

---

# Summary

| Feature | Purpose |
|----------|----------|
| Basic Stored Procedure | Reusable SQL logic |
| Parameters | Dynamic execution |
| Default Parameters | Optional inputs |
| Variables | Store intermediate values |
| Multiple Statements | Execute multiple queries |
| IF...ELSE | Conditional execution |
| IF EXISTS | Efficient existence checking |
| TRY...CATCH | Error handling |
| ERROR_MESSAGE() | Error description |
| ERROR_NUMBER() | SQL Server error code |
| ERROR_LINE() | Line where error occurred |
| ERROR_PROCEDURE() | Procedure name generating the error |

---

# Best Practices

✅ Parameterize Stored Procedures instead of hardcoding values.

✅ Use default parameters whenever appropriate.

✅ Declare variables for intermediate calculations.

✅ Perform data validation or cleansing before generating reports.

✅ Prefer `IF EXISTS` over `COUNT(*)` when checking whether rows exist.

✅ Wrap production Stored Procedures inside `TRY...CATCH`.

✅ Log meaningful error details using SQL Server's built-in error functions.

---

# Interview Questions

## 1. Why do we use Stored Procedures?

**Answer:**

Stored Procedures improve performance, security, code reusability, maintainability, and reduce network traffic by encapsulating business logic within the database.

---

## 2. What are parameterized Stored Procedures?

**Answer:**

They accept input parameters, allowing the same procedure to execute with different values without modifying the SQL code.

---

## 3. Why are default parameters useful?

**Answer:**

Default parameters allow procedures to execute even when optional inputs are omitted, improving flexibility and reducing the need to pass common values repeatedly.

---

## 4. Why do we use variables inside Stored Procedures?

**Answer:**

Variables store intermediate results that can be reused for calculations, conditional logic, printing messages, or passing values between different parts of the procedure.

---

## 5. Why is `IF EXISTS` preferred over `COUNT(*)`?

**Answer:**

`IF EXISTS` stops searching after finding the first matching row, whereas `COUNT(*)` scans all matching rows. Therefore, `IF EXISTS` is generally more efficient when checking for the existence of data.

---

## 6. What is the purpose of TRY...CATCH?

**Answer:**

`TRY...CATCH` provides structured exception handling. Instead of abruptly terminating execution, it captures runtime errors and allows the procedure to log or handle them gracefully.

---

## 7. Which built-in functions can be used inside the CATCH block?

**Answer:**

- `ERROR_MESSAGE()`
- `ERROR_NUMBER()`
- `ERROR_LINE()`
- `ERROR_PROCEDURE()`

These functions provide detailed information about the error that occurred.

---

## 8. Why is error handling important in production Stored Procedures?

**Answer:**

Error handling prevents unexpected failures from crashing the application, provides useful debugging information, enables logging, and allows developers to troubleshoot production issues more effectively.

---

# Key Takeaways

- Stored Procedures encapsulate reusable SQL logic inside the database.
- Parameters make procedures dynamic and reusable.
- Default parameters simplify execution by providing fallback values.
- Variables allow intermediate calculations and message generation.
- `IF...ELSE` enables conditional execution and is commonly used for data validation and cleansing.
- `IF EXISTS` is an efficient way to check whether matching rows exist.
- `TRY...CATCH` ensures robust error handling in production environments.
- SQL Server's built-in error functions (`ERROR_MESSAGE()`, `ERROR_NUMBER()`, `ERROR_LINE()`, and `ERROR_PROCEDURE()`) provide detailed diagnostics for troubleshooting.
```
