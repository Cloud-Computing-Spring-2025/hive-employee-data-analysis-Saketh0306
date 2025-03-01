# HadoopHiveHue
Hadoop , Hive, Hue setup pseudo distributed  environment  using docker compose
# Hive Data Processing - README

## Dataset Details

### **employees.csv**
This dataset contains employee-related information, including department, job role, salary, and project assignment.

**Columns:**
- `emp_id` - Unique employee ID
- `name` - Employee's full name
- `age` - Employee's age
- `job_role` - Designation of the employee
- `salary` - Annual salary of the employee
- `project` - Assigned project (One of: Alpha, Beta, Gamma, Delta, Omega)
- `join_date` - Date when the employee joined
- `department` - Department to which the employee belongs (Used for partitioning)

### **departments.csv**
Contains information about company departments.

**Columns:**
- `dept_id` - Unique department ID
- `department_name` - Name of the department
- `location` - Location of the department

---
## Hive Setup and Data Processing

### **1. Create a Hive Table (Commands Used)**
```sql
CREATE TABLE employees (
    emp_id STRING,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING
)
PARTITIONED BY (department STRING)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS PARQUET;
```

```sql
CREATE TABLE departments (
    dept_id STRING,
    department_name STRING,
    location STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;
```

### **2. Load Data into a Temporary Hive Table**
```sql
CREATE TEMPORARY TABLE temp_employees (
    emp_id STRING,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING,
    department STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;

LOAD DATA LOCAL INPATH '/path/to/employees.csv' 
INTO TABLE temp_employees;
```

### **3. Enable Dynamic Partitioning and Insert Data**
```sql
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT INTO TABLE employees PARTITION (department)
SELECT emp_id, name, age, job_role, salary, project, join_date, department
FROM temp_employees;
```

### **4. Add Partitions Using ALTER TABLE**
```sql
ALTER TABLE employees ADD PARTITION (department='Sales');
ALTER TABLE employees ADD PARTITION (department='IT');
```

---
## Execution Steps

### **1. Copy Dataset to Namenode Container**
```sh
docker cp input_dataset namenode:/tmp/
```

### **2. Move Dataset to HDFS**
```sh
hdfs dfs -mkdir -p /user/hive/warehouse/employees/
hdfs dfs -mkdir -p /user/hive/warehouse/departments/

hdfs dfs -put /tmp/input_dataset/employees.csv /user/hive/warehouse/employees/
hdfs dfs -put /tmp/input_dataset/departments.csv /user/hive/warehouse/departments/
```


## Queries

### **Retrieve all employees who joined after 2015**
```sql
SELECT * FROM employees 
WHERE year(TO_DATE(join_date, 'yyyy-MM-dd')) > 2015;
```

### **Find the average salary of employees in each department**
```sql
SELECT department, AVG(salary) AS avg_salary 
FROM employees 
GROUP BY department;
```

### **Identify employees working on the 'Alpha' project**
```sql
SELECT * FROM employees 
WHERE project = 'Alpha';
```

### **Count the number of employees in each job role**
```sql
SELECT job_role, COUNT(*) AS num_employees 
FROM employees 
GROUP BY job_role;
```

### **Retrieve employees whose salary is above the average salary of their department**
```sql
SELECT e.* 
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS avg_salary 
    FROM employees 
    GROUP BY department
) d ON e.department = d.department
WHERE e.salary > d.avg_salary;
```

### **Find the department with the highest number of employees**
```sql
SELECT department, COUNT(*) AS num_employees
FROM employees 
GROUP BY department 
ORDER BY num_employees DESC 
LIMIT 1;
```

### **Exclude employees with null values in any column**
```sql
SELECT * FROM employees 
WHERE emp_id IS NOT NULL 
AND name IS NOT NULL 
AND age IS NOT NULL 
AND job_role IS NOT NULL 
AND salary IS NOT NULL 
AND project IS NOT NULL 
AND join_date IS NOT NULL 
AND department IS NOT NULL;
```

### **Join employees and departments tables to display employee details along with department locations**
```sql
SELECT e.*, d.location 
FROM employees e 
JOIN departments d 
ON e.department = d.department_name;
```

### **Rank employees within each department based on salary**
```sql
SELECT emp_id, name, department, salary, 
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
```

### **Find the top 3 highest-paid employees in each department**
```sql
SELECT * FROM (
    SELECT emp_id, name, department, salary, 
           RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
) ranked
WHERE salary_rank <= 3;
```
