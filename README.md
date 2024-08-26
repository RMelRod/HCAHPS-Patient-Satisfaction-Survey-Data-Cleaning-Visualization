# HCAHPS Patient Satisfaction Survey Data Cleaning & Visualization
## **1. Project Overview**
**Background:** [The Hospital Value-Based Purchasing (VBP) Program](https://www.cms.gov/medicare/quality/initiatives/hospital-quality-initiative/hospital-value-based-purchasing) is one of several value-based care programs introduced by the Affordable Care Act (2010). Value-based care links provider reimbursement to the quality of care they provide rather than quantity. Under the programs, hospitals receive payment adjustments based on the quality of care the hospitals deliver to patients during inpatient stays. The actual amount of incentive payments a hospital earns is linked to performance measures including clinical outcomes (25%), safety (25%), person and community engagement (25%), efficient and cost reduction (25%).

**Objective:**  To clean, categorize and visualize HCAHPS (Hospital Consumer Assessment of Healthcare Providers and Systems) survey scores. By focusing on top box answers, (survey scores = "Always"/"9-10") for each HCAHPS question, to identify the quality of patient care in 3,000+ hospitals nationwide. 

**Scope:** This project involved importing raw HCAHPS survey data, raw provider cost report data, handling NULL values, de-duplication of records, data standardization, and ensuring data integrity for visualization in Tableau.

**Tools:** Excel, SQL (PostgreSQL), Tableau

**Visualization:** [HCAHPS Patient Survey Analysis](https://public.tableau.com/views/HCAHPSPatientSurveyAnalysis_17232589398030/HCAHPSDashboard?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link).

## 2. Data Acquisition

**Sources**: [hospitals_11_2023.zip_HCAHPS Hospital](https://data.cms.gov/provider-data/archived-data/hospitals), [Hospital Provider Cost Report](https://data.cms.gov/provider-compliance/cost-report/hospital-provider-cost-report/dataquery=%7B%22filters%22%3A%7B%22rootConjunction%22%3A%7B%22label%22%3A%22And%22%2C%22value%22%3A%22AND%22%7D%2C%22list%22%3A%5B%5D%7D%2C%22keywords%22%3A%22%22%2C%22offset%22%3A0%2C%22limit%22%3A10%2C%22sort%22%3A%7B%22sortBy%22%3A%22Fiscal%20Year%20End%20Date%22%2C%22sortOrder%22%3A%22DESC%22%7D%2C%22columns%22%3A%5B%5D%7D)

**Format**: Datasets were downloaded in .csv format and reformatted in Microsoft Excel.

## **3. Loading & Inspection of CSV Data**

After downloading the two CSV files, I extracted the columns I needed utilizing Excel. I renamed the CSVs raw_hcahps_data and raw_hospital_beds. I removed "Not Applicable" and "Not Available" text from numeric columns using the Find and Replace tool in Excel.

## 4. Creating Tables in PostgreSQL & Importing Data 

Using the following query I created tables in PostgreSQL. I also imported the reformatted CSVs raw_HCAHPS_data and raw_hospital_beds files populating the tables.
```
CREATE TABLE IF NOT EXISTS "postgres"."raw_hospital_data".raw_hospital_beds
(
     provider_ccn integer
    ,hospital_name character varying(255)
    ,fiscal_year_begin_date character varying(10)
    ,fiscal_year_end_date character varying(10)
    ,number_of_beds integer
);
CREATE TABLE IF NOT EXISTS "postgres"."raw_hospital_data".raw_HCAHPS_data
(
    facility_id character varying(10),
    facility_name character varying(255),
    address character varying(255),
    city character varying(50),
    state character varying(2),
    zip_code character varying(10),
    county_or_parish character varying(50),
    telephone_number character varying(20),
    hcahps_measure_id character varying(255),
    hcahps_question character varying(255),
    hcahps_answer_description character varying(255),
    hcahps_answer_percent integer,
    num_completed_surveys integer,
    survey_response_rate_percent integer,
    start_date character varying(10),
    end_date character varying(10)
);
```
## **5. Initial Data Exploration: Table I: raw_hospital_beds**

I examined the raw_hospital_beds table. It seemed rather standardized and strightforward for the most part. I only needed the CMS Certification number (CCN), hospital name and bed count corresponding with the most recent dates. There were some NULL bed counts. However, these will be filtered out in Tableau as these hospitals cannot be classified as small medium or large. Reformatting and manipulation of the date column will also be necessary. A provider CCN number must be in a six digit format, so this must be corrected as well. 

## **6. Data Cleaning & Transformation of Table 1: raw_hospital_beds**

### **6.1 Checking for Duplicates**
No duplicates were found using the following query.
```
SELECT
	COUNT (*) AS duplicate_count,
	provider_ccn,
	hospital_name,
	fiscal_year_begin_date,
	fiscal_year_end_date,
	number_of_beds
FROM 
    raw_hospital_data.raw_hospital_beds
GROUP BY 
	provider_ccn,
	hospital_name,
	fiscal_year_begin_date,
	fiscal_year_end_date,
	number_of_beds
HAVING 
    COUNT(*) > 1;
-- 0 rows
```
### **6.2 Reformatting Dates & Provider CCN**
All dates were formatted in a "character varying" format. In order to categorize hospitals as small, medium and large I must extract the most recent bed count available in 2021-2022. Therefore, I reformatted the fiscal_year_end_date, and fiscal_year_end_date column to identify the most recent date that corresponded with the most recent bed count using the TO_DATE function.
All provider ccns were formatted in an integer format consisting of varying digit length. To accurately identify a hospital, it's provider cnn number must be six digits in length. I corrected this using the CAST and LPAD function. 
```
SELECT
	LPAD(CAST(provider_ccn as text),6,'0') AS provider_ccn,
	hospital_name,
	TO_DATE(fiscal_year_begin_date,'MM/DD/YYYY') AS fiscal_year_begin_date,
	TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') AS fiscal_year_end_date,
	number_of_beds
FROM raw_hospital_data.raw_hospital_beds;
```
### **6.3 Extracting Relevant Data**
To extract the most recent bed count, I used a combination of a common table expression (CTE), WINDOW function and PARTITION BY statement. In the CTE I kept all of the previous reformatting and assigned a ROW_NUMBER to the WINDOW function. In the WINDOW function, I partitioned the data by provider_cnn and ordered the hospitals by fiscal_year_end_date in descending order. By assigning a ROW_NUMBER to this window function I was able to identify those hospitals with the most recent data as their nth_row output was the number 1. I then filtered the data where the nth_row is equal to one.
```
WITH hospital_beds_prep AS
(
SELECT
	LPAD(CAST(provider_ccn as text),6,'0') AS provider_ccn,
	hospital_name,
	TO_DATE(fiscal_year_begin_date,'MM/DD/YYYY') AS fiscal_year_begin_date,
	TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') AS fiscal_year_end_date,
	number_of_beds,
	ROW_NUMBER () OVER (PARTITION BY provider_ccn ORDER BY TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') DESC) as nth_row
FROM raw_hospital_data.raw_hospital_beds
)	
SELECT
	provider_ccn,
	hospital_name,
	fiscal_year_begin_date,
	fiscal_year_end_date,
	number_of_beds,
	nth_row
FROM hospital_beds_prep
WHERE nth_row = 1
GROUP BY provider_ccn, hospital_name, fiscal_year_begin_date, fiscal_year_end_date, number_of_beds, nth_row;
```
## **7. Initial Data Exploration Table II: raw_hcahps_data**
I examined the raw_hcahps_data table. As this table was similar to the raw_hospital_beds table, it was apparent I would need to reformat the facility_id. I would also need to standardize this column and rename it provider_ccn. Reformatting of the date column would also be necessary. There were some NULL values. However, these will be filtered out in the final query.

## **8. Data Cleaning & Transformation of Table II: raw_hcahps_data**
### **8.1 Checking for Duplicates**
As I am only interested in the facility_id, facility_name, hcahps_question, hcahps_answer_description, hcahps_answer_percent, num_completed_surveys, survey_response_rate_percent, start_date, and end_date. I decided to check for duplicates on these columns. 
```
SELECT
	COUNT (*) AS duplicate_count,
	facility_id,
	facility_name,
	hcahps_question,
	hcahps_answer_description,
	hcahps_answer_percent,
	num_completed_surveys,
	survey_response_rate_percent,
	start_date,
	end_date 
FROM 
    raw_hospital_data.raw_hcahps_data
GROUP BY 
	facility_id,
	facility_name,
	hcahps_question,
	hcahps_answer_description,
	hcahps_answer_percent,
	num_completed_surveys,
	survey_response_rate_percent,
	start_date,
	end_date
HAVING 
    COUNT(*) > 1;
-- 0 rows
```
### **8.2 Reformatting & Standardization**
All facility_id were formatted in a "character varying" format. As in the raw_hospital_beds dataset I reformatted the facility_id to be 6 digits in length using the LPAD, and CAST functions. I standardized the data and renamed facility_id to provider_ccn. I also reformatted the start_date and end_date to a date format using the TO_DATE function renaming each, start_date_converted and end_date_converted respectively.
```
SELECT
	LPAD(CAST(facility_id AS text),6,'0') AS provider_ccn,
	facility_name,
	TO_DATE(start_date,'MM/DD/YYYY') AS start_date_converted,
	TO_DATE(end_date,'MM/DD/YYYY') AS end_date_converted,
	*
FROM raw_hospital_data.raw_hcahps_data;
```
### **9. Reformatting, Standardization, & Joining Tables I & II**
I joined the following CTE (hospital_beds) with the reformatted raw_hcahps_data table and renamed the raw_hcahps_data table to hcahps.
```
WITH hospital_beds_prep AS
(
SELECT
	LPAD(CAST(provider_ccn as text),6,'0') AS provider_ccn,
	hospital_name,
	TO_DATE(fiscal_year_begin_date,'MM/DD/YYYY') AS fiscal_year_begin_date,
	TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') AS fiscal_year_end_date,
	number_of_beds,
	ROW_NUMBER () OVER (PARTITION BY provider_ccn ORDER BY TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') DESC) AS nth_row
FROM raw_hospital_data.raw_hospital_beds
)
SELECT
	LPAD(CAST(facility_id AS text),6,'0') AS provider_cnn,
	TO_DATE(start_date,'MM/DD/YYYY') AS start_date_converted,
	TO_DATE(end_date,'MM/DD/YYYY') AS end_date_converted,
	*
FROM raw_hospital_data.raw_hcahps_data as hcahps
LEFT JOIN hospital_beds_prep as beds
ON LPAD(CAST(facility_id AS text),6,'0') = beds.provider_ccn
AND beds.nth_row = 1
```
### **9.1 Reformatting, Renaming, Joining & Preparing to Exporting**
I reformatted the remaining columns that needed to be in date format including start_date_converted, and end_date_converted. I then renamed the fiscal_year_begin_date and fiscal_year_end_date to beds_start_report_period, beds_end_report_period, so these columns would be easier to understand when exported. Finally, I filtered out rows where number_of_beds was NULL.
```
WITH hospital_beds_prep AS
(
SELECT
	LPAD(CAST(provider_ccn AS text),6,'0') AS provider_ccn,
	hospital_name,
	TO_DATE(fiscal_year_begin_date,'MM/DD/YYYY') AS fiscal_year_begin_date,
	TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') AS fiscal_year_end_date,
	number_of_beds,
	ROW_NUMBER () OVER (PARTITION BY provider_ccn ORDER BY TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') desc) AS nth_row
FROM raw_hospital_data.raw_hospital_beds
)

SELECT
	LPAD(CAST(facility_id AS text),6,'0') AS provider_ccn,
	TO_DATE(start_date,'MM/DD/YYYY') AS start_date_converted,
	TO_DATE(end_date,'MM/DD/YYYY') AS end_date_converted,
	hcahps.*,
	beds.number_of_beds,
	beds.fiscal_year_begin_date AS beds_start_report_period,
	beds.fiscal_year_end_date AS beds_end_report_period
FROM raw_hospital_data.raw_hcahps_data AS hcahps
LEFT JOIN hospital_beds_prep AS beds
ON LPAD(CAST(facility_id AS text),6,'0')= beds.provider_ccn
AND beds.nth_row = 1
WHERE number_of_beds IS NOT NULL
```
### **9.2 Creation of the Final Table & Exporting**
Using the query below, I created a table called final_tableau_file. I then exported it to a CSV to be used in the tableau visualization.
```
CREATE TABLE "postgres".raw_hospital_data.final_tableau_file2 AS
WITH hospital_beds_prep AS
(
SELECT
	LPAD(CAST(provider_ccn AS text),6,'0') AS provider_ccn,
	hospital_name,
	TO_DATE(fiscal_year_begin_date,'MM/DD/YYYY') AS fiscal_year_begin_date,
	TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') AS fiscal_year_end_date,
	number_of_beds,
	ROW_NUMBER () OVER (PARTITION BY provider_ccn ORDER BY TO_DATE(fiscal_year_end_date,'MM/DD/YYYY') desc) AS nth_row
FROM raw_hospital_data.raw_hospital_beds
)
SELECT
	LPAD(CAST(facility_id AS text),6,'0') AS provider_ccn,
	TO_DATE(start_date,'MM/DD/YYYY') AS start_date_converted,
	TO_DATE(end_date,'MM/DD/YYYY') AS end_date_converted,
	hcahps.*,
	beds.number_of_beds,
	beds.fiscal_year_begin_date AS beds_start_report_period,
	beds.fiscal_year_end_date AS beds_end_report_period
FROM raw_hospital_data.raw_hcahps_data as hcahps
LEFT JOIN hospital_beds_prep AS beds
ON LPAD(CAST(facility_id AS text),6,'0') = beds.provider_ccn
AND beds.nth_row = 1
WHERE hcahps_answer_percent IS NOT NULL
AND num_completed_surveys IS NOT NULL
AND survey_response_rate_percent IS NOT NULL
```
## **10. Visualization of HCAHPS Data in Tableau Dashboard**
### 10.1 Determining Hospital Size
Each hospital was grouped into a cohort consisting of each hospital's size (small, medium, and large) and the state in which it was located. Hospital size was determined based on the number of beds calculated below. A drop down menu was added to filter amongst small, medium and large hospitals in each respective state.
```
IF [Number Of Beds] >= 500 THEN 'Large'
ELSEIF [Number Of Beds] >= 100
AND [Number Of Beds] < 500 THEN 'Medium'
ELSEIF [Number Of Beds] < 100 THEN 'Small'
END
```
### 10.2 Determining Percentage of Patients with Top Box Answers Per Cohort
A top box question contains the key characters of "9-10" or "Always". Therefore, the following if-then statement was used to identify the number of top box questions. These questions corresponded with the number of top box answers selected by patients across all of the HCAHPS questions.
```
IF CONTAINS([Hcahps Question],'Always') OR CONTAINS([Hcahps Question],'9')
THEN 1 ELSE 0
END
```
### 10.3 Determining Top Box Mean Percentage & Delta From Mean Cohort
The mean percentage score for each top box HCAHPS question for each hospital with respect to their size (small, medium, large) and state (the mean cohort) was determined.
To compare each hospital's top box mean scores for each HCAHPS question with other hospitals of the same size and in the same state, I then determined the "Delta From the Mean Cohort" with the calculation below. 
```
[Actual HCAHPS Percent] -
{FIXED[State],[Hospital Size],[Hcahps Answer Description]:AVG([Actual HCAHPS Percent])}
```
### 10.4 Visualizing Overall Hospital Scores Compared to the Mean Cohort
To visualize each hospital's scores per HCAHPS question with respect to the mean cohort, I created the "Cohort Hospital Delta Spread". This compared the quality of patient care for each selected hospital with respect to specific HCAHPS questions compared to the mean cohort.

Visualization of this project can be found at [HCAHPS Patient Survey Analysis](https://public.tableau.com/views/HCAHPSPatientSurveyAnalysis_17232589398030/HCAHPSDashboard?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link).
# **Acknowledgment**
This project was created with the assistance of [Data Wizardry](https://www.youtube.com/@DataWizardry). Special thanks to [Data Wizardry](https://www.youtube.com/@DataWizardry) for their valuable tutorials and resources.
