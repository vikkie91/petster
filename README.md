# Petster
Sample Vet DB  
  
**Scope:**  
- The intention of this README file is to provide a detailed explanation on how I created this DB by taking publicly available datasets and modifying/adding information so that it can be utilized to query and provide the business with actionable insights  
- This would help the reader to understand how I would approach a business issue and use SQL to provide the relevant insights  
- I have used a variety of SQL commands and features including, but not limited to - Joins, Sub-query, CTE, Temporary Table, Windows Functions, Aggregate Functions, Date Functions, String Functions and CASE statements  
- There are three broad classifications of the business questions - Vet Analysis, Pet Analysis and Customer Analysis  
- Under each business question I have provided the SQL query, output and my insights & recommendations  
  
**Methodology:**  
- After downloading the publicly available sample csv files, I added/modified the data to ensure relevancy for my analyses
- In pgadmin (postgres), I created a new DB (petster) and schema (pet)
- Using the below DDL script that I created, I was able to create 5 tables (pets, vets, customers, visits and prescriptions) and import the data from the csv files   

```sql
--DROP TABLES IN BELOW ORDER TO AVOID FK CONSTRAINT ISSUES

DROP TABLE IF EXISTS pet.prescriptions;
DROP TABLE IF EXISTS pet.visits;
DROP TABLE IF EXISTS pet.vets;
DROP TABLE IF EXISTS pet.pets;
DROP TABLE IF EXISTS pet.customers;

--Pets Table

CREATE TABLE pet.pets
(
	pet_id SERIAL PRIMARY KEY,
	owner_id int NOT NULL,
	pet_name text NOT NULL,
	pet_type text,
	breed text,
	colour text,
	sex char(1),
	neutered char(1),
	microchip char(1)
);

copy pet.pets(owner_id,pet_name,pet_type,breed,colour,sex,neutered,microchip) FROM '...\pet_data_updated.csv' DELIMITER ',' CSV HEADER;


--Customers Table

CREATE TABLE pet.customers
(
	customer_id SERIAL PRIMARY KEY,
	first_name text NOT NULL,
	last_name text NOT NULL,
	address text,
	city text,
	province char(2),
	postal text,
	phone text,
	email text
);

copy pet.customers(first_name,last_name,address,city,province,postal,phone,email) FROM '...\customer_data.csv' DELIMITER ',' CSV HEADER;


--Visits Table

CREATE TABLE pet.visits
(
	visit_id SERIAL PRIMARY KEY,
	pet_id int NOT NULL,
	vet_id int NOT NULL,
	visit_date date NOT NULL,
	charges decimal(5,2),
	discount decimal(5,2),
	offer_desc text
);

copy pet.visits(pet_id,vet_id,visit_date,charges,discount,offer_desc) FROM '...\visits.csv' DELIMITER ',' CSV HEADER;

--Vets Table

CREATE TABLE pet.vets
(
	vet_id SERIAL PRIMARY KEY,
	vet_first_name text NOT NULL,
	vet_last_name text NOT NULL,
	province char(2) NOT NULL,
	vet_license_no int NOT NULL,
	salary int NOT NULL
);

copy pet.vets(vet_first_name,vet_last_name,province,vet_license_no,salary) FROM '...\vets.csv' DELIMITER ',' CSV HEADER;

--Prescriptions Table

CREATE TABLE pet.prescriptions
(
	visit_id int NOT NULL,
	prescription text NOT NULL,
	qty int NOT NULL,
	charges decimal(6,2) NOT NULL
);

copy pet.prescriptions(visit_id,prescription,qty,charges) FROM '...\prescriptions.csv' DELIMITER ',' CSV HEADER;

ALTER TABLE pet.pets
    ADD CONSTRAINT fk_pets_customers FOREIGN KEY (owner_id) REFERENCES pet.customers (customer_id);

ALTER TABLE pet.prescriptions
    ADD CONSTRAINT fk_prescriptions_visits FOREIGN KEY (visit_id) REFERENCES pet.visits (visit_id);

ALTER TABLE pet.visits
    ADD CONSTRAINT fk_visits_pets FOREIGN KEY (pet_id) REFERENCES pet.pets (pet_id);

ALTER TABLE pet.visits
    ADD CONSTRAINT fk_visits_vets FOREIGN KEY (vet_id) REFERENCES pet.vets (vet_id);

```
  
   
## ER Diagram

 ![ER Diagram](https://user-images.githubusercontent.com/99361886/157273292-3178b14d-a865-4cf8-8afe-a5d5ae55e15a.png)

## Vet Analysis  

Q1) Which vet has the highest vists and largest customer base?
  
```sql  
SELECT
	concat(ve.vet_first_name,' ',ve.vet_last_name)as Vet_Name,
	count(vi.visit_id) as "Total Visits",
	count(distinct cu.customer_id) as "Customer Base"
FROM
	pet.visits as vi
INNER JOIN
	pet.vets as ve
ON
	ve.vet_id = vi.vet_id
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
INNER JOIN
	pet.customers  as cu
ON
	pe.owner_id = cu.customer_id
	
GROUP BY
	1
ORDER BY
	2 DESC,3 DESC
LIMIT 1
;
  
```  
### Output  
![Vet Analysis - Q1](https://user-images.githubusercontent.com/99361886/157279419-a32d64ce-03b0-4e93-925b-4a03e800af31.png)  
  
### Insights & Action  
- We can see, even for the vet with the highest number of visits, is only 36 for 2 years which roughly translates to one visit every 3 week
- The business has the opportunity to follow up and send reminders for annual checkups and vaccinations to existing customers to increase visits
- The business also has opportunity to follow up with potential customers (who signed up, visited website) but are yet to make a visit  


Q2) The Ops team would like to know how efficient each vet is in retaining customers. For this they would like to know the new vs repeat customers by vet (count & percentage)
  
```sql
SELECT 
	vet_id,
	vet_first_name||' '||vet_last_name as Vet_Name,
	count(case when visit_rank = 1 then 1 else null end)  as New_Customer_Count,
	concat(round(100*cast(count(case when visit_rank = 1 then 1 else null end) as decimal)/(count(case when visit_rank = 1 then 1 else null end) + count(case when visit_rank > 1 then 1 else null end)),2),'%')  as New_Customer_Pct,
	count(case when visit_rank > 1 then 1 else null end)  as Returning_Customers,
	concat(round(100*cast(count(case when visit_rank > 1 then 1 else null end) as decimal)/(count(case when visit_rank = 1 then 1 else null end) + count(case when visit_rank > 1 then 1 else null end)),2),'%')  as Returning_Customer_Pct
	FROM
(
SELECT
	vi.visit_id,
	ve.vet_id,
	ve.vet_first_name,
	ve.vet_last_name,
	row_number() over (partition by vi.vet_id,cu.customer_id order by vi.visit_date) as visit_rank
FROM
	pet.visits as vi
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
INNER JOIN
	pet.customers as cu
ON
	pe.owner_id = cu.customer_id
INNER JOIN
	pet.vets as ve
ON
	vi.vet_id = ve.vet_id
	) as rank_table
GROUP BY
	1,2
ORDER BY
	vet_id;
  
```  
### Output  

![Vet Analysis - Q2](https://user-images.githubusercontent.com/99361886/157279461-d0ffe96d-f00b-4e45-aec5-3ee7f766dcdc.png)  

### Insights & Action
- Here we see that hardly 1 in 5 customers make a subsequent visit. This is across the board with some vets having slightly better return rate than others.
- This gives us the opportunity to send a questionnaire to first-time customers to rate their experience and understand any issues and use the feedback to coach/train vets on customer experience. Through the questionnaire we can also understand if the low return rate is due to any software bugs or issues preventing them from booking or affecting their site/app experience  

  
Q3) The Ops team would also like to know, for each vet, how many customers came back to them and how many went to another vet for their subsequent appointment?
  
```sql  
WITH visit_return as (

SELECT
	vi.visit_id,
	ve.vet_id,
	ve.vet_first_name,
	ve.vet_last_name,
	count(cast(concat(vi.pet_id,vi.vet_id) as int)) over (partition by vi.pet_id order by vi.visit_date) as Returning_visits,
	rank() over (partition by vi.pet_id,vi.vet_id order by vi.visit_date) as same_vet_return_count
FROM
	pet.visits as vi
INNER JOIN
	pet.vets as ve
ON
	vi.vet_id = ve.vet_id)
	
SELECT
	vet_id,
	concat(vet_first_name,' ',vet_last_name),	
	count(visit_id) as Total_Returned_visits,
	count(case when returning_visits > 1 and same_vet_return_count = 1 then 1 else null end) as Switched_Return_visits,
	count(case when returning_visits > 1 and same_vet_return_count <> 1 then 1 else null end) as Retained_Returned_visits
	
FROM
	visit_return
WHERE
	returning_visits = 2
GROUP BY
	1,2;
```  
### Output  
![Vet Analysis - Q3](https://user-images.githubusercontent.com/99361886/157279503-94fa5b77-7c29-4140-8b9d-a9f1a7e8561e.png)

### Insights & Action  
- As an example, we see that out of the 8 patients who made their subsequent visit with Dr.Emma Glacer, only 2 of them had made their initial visit with her. The remaining 6 had their inital visit with another vet
- Since switched visits are much higher than retained visits for most vets, this gives us the opportunity to understand if there is an issue of vet availability during re-booking or any other issue preventing them from going back to the same vet  

Q4) What is the Total Revenue (visit revenue + prescription revenue – discount) brought by each vet? 
  
```sql
DROP TABLE IF EXISTS prescriptions_agg;

CREATE TEMPORARY TABLE prescriptions_agg as
SELECT
	visit_id,
	sum(qty) as Qty,
	sum(charges) as Charges
FROM
	pet.prescriptions
GROUP BY
	1;
	
SELECT
	ve.vet_id,
	ve.vet_first_name||' '||ve.vet_last_name as Vet_Name,
	coalesce(sum(vi.charges),0) as Visit_Charges,
	coalesce(sum(vi.discount),0) as Discounts,
	coalesce(sum(pr.charges),0) as Prescription_Charges,
	coalesce(sum(vi.charges),0) - coalesce(sum(vi.discount),0) + coalesce(sum(pr.charges),0) as Total_Revenue
FROM
	pet.visits as vi
LEFT JOIN
	prescriptions_agg as pr
ON
	vi.visit_id = pr.visit_id
INNER JOIN
	pet.vets as ve
ON
	vi.vet_id = ve.vet_id
GROUP BY
	1,2
ORDER BY
	6 DESC;
```  
### Output  
![Vet Analysis - Q4](https://user-images.githubusercontent.com/99361886/157279578-285a459a-078c-47b1-93ae-4bbf0bf7c002.png)  

### Insights & Action  
- The vet with highest total revenue has ~20% of his revenue attributed to prescription sales
- The business needs to ensure other vets are using the available resources to prescribe meds (when necessary)
- The business can also get feedback from vets if they are facing any issues with respect to availability of stock, brand, etc.  


      
## Pet Analysis  

Q1) What are the top 3 dog and cat breeds (Show Revenue and % of Total Pet Type (i.e. dog,cat) Revenue)  
```sql
WITH breed_wise_revenue as (
SELECT
	pe.breed,
	pe.pet_type,
	coalesce(sum(vi.charges),0) - coalesce(sum(vi.discount),0) + coalesce(sum(pr.charges),0) as Total_Breed_Revenue,
	sum(coalesce(sum(vi.charges),0) - coalesce(sum(vi.discount),0) + coalesce(sum(pr.charges),0))over(partition by pet_type) as Total_Pet_Revenue,
	rank() over (partition by pet_type order by coalesce(sum(vi.charges),0) - coalesce(sum(vi.discount),0) + coalesce(sum(pr.charges),0) DESC) as Revenue_Rank
FROM
	pet.visits as vi
LEFT JOIN
	prescriptions_agg as pr
ON
	vi.visit_id = pr.visit_id
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
GROUP BY
	1,2
ORDER BY 4)

SELECT
	pet_type,
	breed,
	revenue_rank,
	total_breed_revenue,
	total_pet_revenue,
	round(100*total_breed_revenue/total_pet_revenue,2)||'%' as Share_of_Revenue
	
	
FROM
	breed_wise_revenue
WHERE
	Revenue_Rank in (1,2,3)
ORDER BY
	1,3
	;
```  
### Output  
![Pet Analysis - Q1](https://user-images.githubusercontent.com/99361886/157279934-74a6b565-d9ad-4208-9c5f-4b1806339c5f.png)  

### Insights & Action  
- We see that the Top 3 cat breeds contribute to ~43% of the total cat sales which is significantly higher than Top 3 dog breed revenue share (~10%)
- Given the high share of total sales, the business can review to see if all vets have the capability to treat these cat breeds ,and if not, hire someone who specializes in these breeds  

  
Q2) Which breed (cat or dog) has highest prescription drug issuance?  

```sql
SELECT
	pe.breed,
	pe.pet_type,
	sum(pr.qty) as Qty,
	sum(pr.charges) as Charges
FROM
	pet.visits as vi
INNER JOIN
	pet.prescriptions as pr
ON
	vi.visit_id = pr.visit_id
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
GROUP BY
	1,2
ORDER BY
	4 DESC
LIMIT 1;
```  
### Output  
![Pet Analysis - Q2](https://user-images.githubusercontent.com/99361886/157279968-3b1517e8-5703-4c1f-96eb-f6ece3150c19.png)  

### Insights & Action  
- The business needs to ensure that the meds which are prescribed most often for beagles are in stock  

  
Q3a) The marketing team wants to understand if neutering has an impact on frequency of visits and prescriptions. If so they would like to use this data to send an email campaign to list of customers with non-neutered pets asking them to get their pets neutered with their vets  

```sql
SELECT 
	pe.neutered,
	count(distinct pe.pet_id) as No_of_Pets,
	round(count(vi.visit_id)::NUMERIC/count(distinct pe.pet_id),2)  as No_Of_Visits_per_pet,
	round(sum(pr.charges)/count(distinct pe.pet_id),2)  as Prescrption_Charges_per_Pet
	
FROM
	pet.visits as vi
LEFT JOIN
	prescriptions_agg as pr
ON
	vi.visit_id = pr.visit_id
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
INNER JOIN
	pet.customers as cu
ON
	pe.owner_id = cu.customer_id
GROUP BY
	1;
```  
### Output  
![Pet Analysis - Q3a](https://user-images.githubusercontent.com/99361886/157280313-bc1a3c08-f534-4389-8008-33bd78f03edc.png)  

### Insights & Action  
- Though it is not very significant, pets that are not neutered tend to have more number of visits and need of prescription drugs 


Q3b) List of customer email_ids with non-neutered pets  
```sql
SELECT 
	cu.email as Customer_email,
	pe.neutered as Neutered_status
FROM
	pet.customers as cu
INNER JOIN
	pet.pets as pe
ON
	cu.customer_id = pe.owner_id
WHERE
	pe.neutered = 'N';
```  
### Output (First 10 rows)  
![Pet Analysis - Q3b](https://user-images.githubusercontent.com/99361886/157280343-e54da1e0-9d5f-4501-970e-8078711fb737.png)  

### Insights & Action  
- As a next step, the business can find out where these customers reside and in the email provide the address of the closest vets/clinics that provide neutering services


     
## Customer Analysis  

Q1) How many new customers in 2020 returned in 2021?
  
```sql  
WITH new_return_customers as (
SELECT
	cu.customer_id,
	extract (year from (min(visit_date))) ,
	extract (year from (max(visit_date)))as Year_of_Visit, 
	case when extract (year from (max(visit_date)))>extract (year from (min(visit_date))) then 1 else null end as Returning_Customer,
	case when extract (year from (max(visit_date)))=extract (year from (min(visit_date))) then 1 else null end as New_Customer
FROM
	pet.customers as cu
INNER JOIN
	pet.pets as pe
ON
	cu.customer_id = pe.owner_id
INNER JOIN
	pet.visits as vi
ON
	pe.pet_id = vi.pet_id
GROUP BY
	1
ORDER BY
	2 DESC,3)


SELECT
	Year_of_Visit,
	count(nr.Returning_Customer) as Returning_Customers,
	count(nr.New_Customer) as New_Customers
	
FROM
	new_return_customers as nr
	
GROUP BY
	1
	
;
```  
### Output  
![Customer Analysis - Q1](https://user-images.githubusercontent.com/99361886/157280449-04b5eeb3-6a1a-4d89-b32d-042ea8e84523.png)  

### Insights & Action  
- 52 out of 106 (52+54) customers who made a visit in 2020 came back in 2021  
- Business can send promotional offers to the remaining 51% incentivizing them to make a visit in 2021 

  

Q2) Which Provinces have prescription charges that are above National average?
  
```sql  
With provincial_prescriptions as (

SELECT 	
	cu.province,
	sum(pr.charges) as Pres_Charges
FROM 
	pet.visits as vi
INNER JOIN
	pet.prescriptions as pr
ON
	vi.visit_id = pr.visit_id
INNER JOIN
	pet.pets as pe
ON
	vi.pet_id = pe.pet_id
INNER JOIN
	pet.customers as cu
ON
	pe.owner_id = cu.customer_id
GROUP BY
	1
)

SELECT 	
	province,
	pres_charges,
	'Above Average' as Prescriptions
FROM 
	provincial_prescriptions
GROUP BY
	1,2
HAVING
	sum(pres_charges)>(SELECT avg(pres_charges) FROM provincial_prescriptions)

	;
```  
### Output  
![Customer Analysis - Q2](https://user-images.githubusercontent.com/99361886/157280516-eeca1b61-a77f-49a4-bd9a-e877bbef3858.png)  

### Insights & Action  
- Business needs to ensure adequate stock of meds in these 3 provinces



	
	
Q3) What % of customers have more than 1 pet and how is their average revenue compared to customer with one pet?
  
```sql
SELECT
	count(Single_Pet_Owners) as Single_Pet_Owners,
	round(100*count(Single_Pet_Owners)::NUMERIC/count(distinct owner_id),2) as single_pct,
	count(Multiple_Pet_Owners) as Multiple_Pet_Owners,
	round(100*count(Multiple_Pet_Owners)::NUMERIC/count(distinct owner_id),2) as multiple_pct
FROM 
(
SELECT
	pe.owner_id,
	pe.pet_id,
	count(pe.pet_id) over (partition by pe.owner_id) as pet_counts,
	row_number() over (partition by pe.owner_id) as row_counts,
	case when count(pe.pet_id) over (partition by pe.owner_id)=1 then 1 else null end as Single_Pet_Owners,
	case when count(pe.pet_id) over (partition by pe.owner_id)>1 and row_number() over (partition by pe.owner_id)=1 then 1 else null end as Multiple_Pet_Owners
FROM
	pet.pets as pe
ORDER BY
	1
) as pet_owners
```  
### Output  
![Customer Analysis - Q3](https://user-images.githubusercontent.com/99361886/157280551-1ca0fb39-e0f2-45b9-9b75-59038cc7012b.png)  

### Insights & Action  
- Though owners with multiple pets only account for 16% of total customers it does not make them less important. As a next step, we can try to understand the average lifetime value of single vs multiple pet owners from the two years of data and see what is the delta in revenue they generate over single pet owners
- If multiple pet ownership is highly profitable for the business, it can incentivize single pet owners to get more pets by offering discounts on the subsequent pet



