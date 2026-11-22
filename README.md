-- GreenTech Manufacturing Evaluating Production Bottlenecks Through Downtime Analysis
--Data Assessment and Understanding the data structure using t-sequel to query the database
SELECT * FROM downtime_factors;
SELECT * FROM line_downtime;
SELECT * FROM line_productivity_batches
SELECT * FROM products

-- Number of rows
SELECT COUNT(*) AS Rows_downtime_factor FROM downtime_factors;
SELECT COUNT(*) AS Rows_line_downtime FROM line_downtime;
SELECT COUNT(*) AS Rows_line_productivity_batches FROM line_productivity_batches;
SELECT COUNT(*) AS Rows_products FROM products;

-- Check for Nulls
SELECT COUNT(*) AS Factor1_Null_Count FROM line_downtime
WHERE Factor_1 IS NULL;

SELECT COUNT(*) AS Factor2_Null_Count FROM line_downtime
WHERE Factor_2 IS NULL;

SELECT COUNT(*) AS all_factors_null_count from line_downtime
WHERE COALESCE(Factor_1, Factor_2,Factor_3,Factor_4,Factor_5,Factor_6,Factor_7,Factor_8,Factor_9,Factor_10,Factor_11,Factor_12,Factor_13) IS NULL

--CTE (Common Table Expression) Start Date repitition in "Date" and "Start_Time" Columns
WITH Start_dates AS (
      SELECT Date, Cast(Start_Time As Date) As Start_Date 
      From line_productivity_batches)
SELECT * FROM Start_dates
WHERE Date != Start_Date;

-- DATA Cleaning and Transformation 
-- Handle null line downtime (Unpivot table and ensure consistency in factor name in relation to downtime factors table

CREATE VIEW downtimes AS
SELECT Batch_ID,REPLACE(Factor,'Factor_', '') AS Factor_ID, Minutes FROM line_downtime
UNPIVOT(Minutes FOR Factor IN (Factor_1, Factor_2,Factor_3,Factor_4,Factor_5,Factor_6,Factor_7,Factor_8,Factor_9,Factor_10,Factor_11,Factor_12,Factor_13)) AS UnpivotDowntimes;

SELECT * FROM downtimes
ORDER BY Batch_ID
-- New Batch production table(Date as Start Date, Extract time from start_time and end time

CREATE VIEW batch_pd AS
     SELECT Date AS Start_Date, Product_ID, Batch_ID, Operator, CAST(End_Time AS Date) AS End_Date,
     CAST(Start_Time AS Time) AS Start_Time, Cast(End_Time As Time) AS End_Time, Planned_Min_Batch_Hours,
     DATEDIFF(hour, Start_Time, End_Time) AS Actual_Duration,
     DATEDIFF(hour, Start_Time, End_Time)- Planned_Min_Batch_Hours AS Extra_time_Hr
     FROM line_productivity_batches


     SELECT * FROM batch_pd 
  
  -- Validating downtime minutes against planned production time and reported start/end time
  SELECT Batch_pd.Batch_ID, SUM(Minutes) AS accounted_delay_minutes, Extra_Time_Hr FROM batch_pd
  JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
  GROUP BY batch_pd.Batch_ID, Extra_Time_Hr;

  -- ANALYSIS
  -- Downtime key factors

SELECT Factor_Name, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_Mins FROM downtimes
JOIN Downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
GROUP BY Factor_Name
Order BY Frequency Desc;

-- Operator vs Non Operator Errors

SELECT
    CASE Operator_Error
    WHEN 1 THEN 'Yes'
    WHEN 0 Then 'No'
END AS Operator_Error,
COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_Mins FROM downtimes
JOIN Downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
GROUP BY Operator_Error

-- Downtime Operator Errors
SELECT Factor_Name, Description, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_Mins FROM downtimes
JOIN Downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
WHERE Operator_Error = 1
GROUP BY Factor_Name, Description
ORDER BY SUM(Minutes) DESC;

-- Downtime Non Operator Errors

SELECT Factor_Name, Description, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_Mins FROM downtimes
JOIN Downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
WHERE Operator_Error = 0
GROUP BY Factor_Name, Description
ORDER BY SUM(Minutes) DESC;

-- Products and downtime frequency & delays (mins)
SELECT batch_pd.Product_ID,Product_Name, COUNT(downtimes.Batch_ID) AS Frequency, SUM(Minutes) AS Delay_Mins FROM batch_pd
JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
JOIN products ON batch_pd.Product_ID = products.Product_ID
GROUP BY batch_pd.Product_ID, Product_Name

-- How many factors are involved in each product downtime?
SELECT batch_pd.Product_ID, Product_Name, COUNT(DISTINCT Factor_ID) AS Distinct_Factor FROM batch_pd
JOIN products ON batch_pd.Product_ID = products.Product_ID
JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
GROUP BY batch_pd.Product_ID, Product_Name


    --TOP 5 Factors for Product 1 downtime
SELECT TOP 5 Factor_Name, SUM(Minutes) AS Prd001_Delay_Mins FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD001'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC

 --TOP 5 Factors for Product 2 downtime
SELECT TOP 5 Factor_Name, SUM(Minutes) AS Prd002_Delay_Mins FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD002'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC

 --TOP 5 Factors for Product 3 downtime
 SELECT TOP 5 Factor_Name, SUM(Minutes) AS Prd003_Delay_Mins FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD003'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC

--TOP 5 Factors for Product 4 downtime
SELECT TOP 5 Factor_Name, SUM(Minutes) AS Prd004_Delay_Mins FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD004'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

--Production Lead Operators
SELECT Operator, Count(Batch_ID) AS Number_of_Batches, COUNT(Distinct Product_ID) as Number_of_Products
FROM batch_pd
GROUP BY Operator

-- Production Lead operator and Downtime Duration, Percentage delayed Batches
SELECT 
Operator,
COUNT(DISTINCT batch_pd.Batch_ID) AS Total_Batches,
COUNT(DISTINCT downtimes.Batch_ID) AS Number_of_delayed_batches,
COUNT(downtimes.Batch_ID) As Number_of_downtimes,
SUM(Minutes) AS Delay_Mins,
CAST((COUNT(DISTINCT downtimes.Batch_ID)*100.0)/(COUNT(DISTINCT batch_pd.Batch_ID)) AS DECIMAL(10,2)) 
AS Percentage_delayed_batches
FROM batch_pd
LEFT JOIN downtimes ON batch_pd.Batch_ID = downtimes.batch_ID
GROUP BY Operator
ORDER BY Percentage_delayed_batches DESC;

-- Factors causing downtime for top 3 lead operator with the most delay duration
-- 1. Paul

SELECT Factor_Name,
     SUM(Minutes) AS Delay_Mins 
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'Paul'
     GROUP BY Factor_Name
     ORDER BY SUM(Minutes) DESC;

-- 2. James

SELECT Factor_Name,
     SUM(Minutes) AS Delay_Mins 
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'James'
     GROUP BY Factor_Name
     ORDER BY SUM(Minutes) DESC;


-- 3. Emily

SELECT Factor_Name,
     SUM(Minutes) AS Delay_Mins,
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'Emily'
     GROUP BY Factor_Name
     ORDER BY SUM(Minutes) DESC;

-- Factors causing downtime for the  top 3 operators with the most percentage
-- delayed batch
-- 1. Linda
SELECT Factor_Name,
     COUNT(DISTINCT downtimes.Batch_ID) AS Number_of_delayed_batches,
     SUM(Minutes) AS Delay_Mins
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'Linda'
     GROUP BY Factor_Name
     ORDER BY Number_of_delayed_batches;

     

-- 2. Sophia
SELECT Factor_Name,
     COUNT(DISTINCT downtimes.Batch_ID) AS Number_of_delayed_batches,
     SUM(Minutes) AS Delay_Mins
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'Sofia'
     GROUP BY Factor_Name
     ORDER BY Number_of_delayed_batches;

-- 3. Rita
SELECT Factor_Name,
     COUNT(DISTINCT downtimes.Batch_ID) AS Number_of_delayed_batches,
     SUM(Minutes) AS Delay_Mins
     From downtime_factors
     JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
     JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
     WHERE Operator = 'Rita'
     GROUP BY Factor_Name
     ORDER BY Number_of_delayed_batches

    -- Production scheduling for specific lead operator
    SELECT [Start_Date],
        DATEPART(HOUR, Start_Time) AS Start_Hour,
        Product_ID
        FROM batch_pd
        WHERE Operator = 'Paul' AND Start_Date IN(
        SELECT Start_Date FROM batch_pd
        WHERE Operator = 'Paul'
        GROUP BY Start_Date
        HAVING COUNT(*) > 1)
        ORDER BY Start_Hour DESC
