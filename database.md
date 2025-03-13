# This file cotains the answers for the Database section of the technical test.


## Q1
**How would you design a database to capture the type of information and data in cell-count.csv? Imagine that you’d have hundreds of projects, thousands of samples and various types of analytics you’d want to perform, including the example analysis of responders versus non-responders comparisons above. Please provide a rough prototype schema.**

I would use a relational database to store information like the samples found in cell count at a large scale. This database design decision would allow for the storage of the structured and relational data found in the dataset. Each project can have multiple subjects, which can have multiple samples, and each sample can be associated with multiple immune cell populations and their respective counts. These clear relationships can be easily and effectively modeled with a relational database like postgreSQL.

Furthermore, unlike storing all the data in the format shown in the csv file, using a relational database can save on data storage costs as less data needs to be stored due to less duplicate information as a result of modelling relationships.

On top of that, this can allow us to define constraints on the data using the "CHECK" constraint in postgreSQL or other similar constraints.

This design ensures analysis with large amounts of data if we use good indexing and well-designed queries.

Here's how I expect the schema to look like:
```
CREATE TABLE Project (
    project_id SERIAL PRIMARY KEY,
    project_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT
);
```

```
CREATE TABLE Subject (
    subject_id SERIAL PRIMARY KEY,
    project_id INT NOT NULL REFERENCES Project(project_id),
    condition VARCHAR(50) CHECK (sample_type IN ('melanoma', 'healthy', 'lung')), -- null values allowed; can change the check
    age INT,
    sex CHAR(1),
    treatment INT,
    response CHAR(1) CHECK (response IN ('y', 'n'))  -- null values allowed
);
```

```
CREATE TABLE Sample (
    sample_id SERIAL PRIMARY KEY,
    subject_id INT NOT NULL REFERENCES Subject(subject_id),
    sample_type VARCHAR(50) CHECK (sample_type IN ('PBMC', 'tumor')), -- null values allowed; can change the check
    time_from_treatment_start FLOAT(24)
);
```

```
CREATE TABLE PopulationCounts (
    popcount_id SERIAL PRIMARY KEY,
    sample_id INT REFERENCES Sample(sample_id),
    cell_name VARCHAR(100) NOT NULL,
    raw_count INT NOT NULL CHECK (raw_count >= 0),
    relative_frequency FLOAT NOT NULL CHECK (relative_frequency BETWEEN 0 AND 1)
);
```

## Q2
**What would be some advantages in capturing this information in a database?**

As described above, capturing this information in a relational database follows from the clear relationships in the data that can be modelled to reduce the size of the data while formally specifying how different entities relate to each other. We can also specify constraints on certain fields to ensure correctness when loading data into the database.

## Q3
**Based on the schema you provide in (1), please write a query to summarize the number of subjects available for each condition.**


```
SELECT condition, COUNT(*) AS subject_count
FROM Subject
GROUP BY condition
ORDER BY subject_count DESC; -- can change order if necessary
```

## Q4
**Please write a query that returns all melanoma PBMC samples at baseline (time_from_treatment_start is 0) from patients who have treatment tr1.**

```
SELECT s.sample_id, s.subject_id, s.sample_type, s.time_from_treatment_start
FROM Sample s
JOIN Subject subj ON s.subject_id = subj.subject_id
WHERE subj.condition = 'melanoma'
    AND subj.treatment = 1
    AND s.sample_type = 'PBMC'
    AND s.time_from_treatment_start = 0;
```

## Q5
**Please write queries to provide these following further breakdowns for the samples in (4):**
**a. How many samples from each project**

```
WITH melanoma_pbmc_baseline AS (
    SELECT s.sample_id, subj.subject_id, subj.project_id
    FROM Sample s
    JOIN Subject subj ON s.subject_id = subj.subject_id
    WHERE subj.condition = 'melanoma'
        AND subj.treatment = 1
        AND s.sample_type = 'PBMC'
        AND s.time_from_treatment_start = 0
)
SELECT p.project_name, COUNT(mp.sample_id) AS sample_count
FROM melanoma_pbmc_baseline mp
JOIN Project p ON mp.project_id = p.project_id
GROUP BY p.project_name
ORDER BY sample_count DESC;
```

**b. How many responders/non-responders**

```
WITH melanoma_pbmc_baseline AS (
    SELECT s.sample_id, subj.subject_id, subj.response
    FROM Sample s
    JOIN Subject subj ON s.subject_id = subj.subject_id
    WHERE subj.condition = 'melanoma'
      AND subj.treatment = 1
      AND s.sample_type = 'PBMC'
      AND s.time_from_treatment_start = 0
)
SELECT mp.response,COUNT(mp.sample_id) AS sample_count
FROM melanoma_pbmc_baseline mp
GROUP BY mp.response
ORDER BY sample_count DESC;
```

**c. How many males, females**

```
WITH melanoma_pbmc_baseline AS (
    SELECT s.sample_id, subj.subject_id, subj.sex
    FROM Sample s
    JOIN Subject subj ON s.subject_id = subj.subject_id
    WHERE subj.condition = 'melanoma'
        AND subj.treatment = 1
        AND s.sample_type = 'PBMC'
        AND s.time_from_treatment_start = 0
)
SELECT mp.sex, COUNT(mp.sample_id) AS sample_count
FROM melanoma_pbmc_baseline mp
GROUP BY mp.sex
ORDER BY sample_count DESC;
```
