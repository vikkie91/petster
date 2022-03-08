# Petster
Sample Vet based DB  
  
**Scope:**  
- The intention of this README file is to provide a step-by-step tutorial on how I created this DB by taking publicly available datasets and modifying/adding additional information so that it can be utilized to query and provide the business with actionable insights  
- This would help the reader to understand how I would approach a business issue and use SQL to provide the relevant insights  
- I have used a variety of SQL commands and features including, but not limited to - Joins, Sub-query, CTE, Temporary Table, Windows Functions, Aggregate Functions, Date Functions, String Functions and CASE statements  
- There are three broad classifications of the business questions - Vet Analysis, Pet Analysis and Customer Analysis  
- Under each business question I have provided notes for additional info (wherever necessary) and the SQL script I created to solve the answer the question  
  
**Methodology:**  
- After downloading the publicly available sample csv files, I added/modified the data to ensure relevancy for my analyses
- In pgadmin, I created a new DB (petster) and schema (pet)
- Using the below DDL script that I created, I was able to create 5 tables (pets, vets, customers, visits and prescriptions) and import the data from the csv files   

```sql
--DROP TABLES IN BELOW ORDER TO AVOID FK CONSTRAINT ISSUES

DROP TABLE IF EXISTS pet.prescriptions;
DROP TABLE IF EXISTS pet.visits;
DROP TABLE IF EXISTS pet.vets;
DROP TABLE IF EXISTS pet.pets;
DROP TABLE IF EXISTS pet.customers;

--Pets Table

DROP TABLE IF EXISTS pet.pets;

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

DROP TABLE IF EXISTS pet.customers;

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

DROP TABLE IF EXISTS pet.visits;

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

DROP TABLE IF EXISTS pet.vets;

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

DROP TABLE IF EXISTS pet.prescriptions;

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
   
     
## Pet Analysis  
   
     
## Customer Analysis


