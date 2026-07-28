# SQL Scripts: Schema Setup & Verification

This reference contains SQL scripts used to populate sample data on the source database and verify schema integrity post-migration.

---

## 1. Source Database Sample Schema Setup

Run on `postgres17-source` prior to migration:

```sql
-- Connect to database
\c company_db;

-- Create Schema
CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL,
    location VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    hire_date DATE NOT NULL,
    salary NUMERIC(10, 2) NOT NULL,
    department_id INT REFERENCES departments(department_id)
);

-- Insert Sample Data
INSERT INTO departments (department_name, location) VALUES
('DevOps & Infrastructure', 'Seattle'),
('Cloud Engineering', 'New York'),
('Database Administration', 'Austin');

INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id) VALUES
('Asbah', 'Rehman', 'asbah.rehman@example.com', '2023-01-15', 125000.00, 1),
('John', 'Doe', 'john.doe@example.com', '2023-03-20', 110000.00, 2),
('Jane', 'Smith', 'jane.smith@example.com', '2023-06-10', 118000.00, 1),
('Michael', 'Brown', 'michael.brown@example.com', '2024-02-01', 105000.00, 3);
```

---

## 2. Post-Migration Verification Queries

Run on `postgres18-target` after `pg_restore`:

```sql
-- Check Table Row Counts
SELECT 'departments' AS table_name, COUNT(*) FROM departments
UNION ALL
SELECT 'employees' AS table_name, COUNT(*) FROM employees;

-- Join Verification Query
SELECT 
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    e.email,
    d.department_name,
    d.location
FROM employees e
JOIN departments d ON e.department_id = d.department_id
ORDER BY e.employee_id;

-- Check Database Engine Version
SELECT version();
```
