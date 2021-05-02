After seeing what happens when we read and write data from a table with a simple Primary key, let's see what a compound (or composite) Primary key is and why it matters. 

## Compound/Composite Primary Key and Clustering Key

You should be in CQL Shell after completing the previous step. 

What happens if we want to query our data by pet_chip_id but also by time? That is if our query is:

`SELECT * from heartrate_v1 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23 AND time >='2019-03-04 07:01:00' AND time <='2019-03-04 07:02:00';`{{execute}}

Try the above query. It fails. We can define the Primary Key to include more than one column, in which case it is called a Composite or Compound key.

Let’s create the following table:
`CREATE TABLE heartrate_v2 (
   pet_chip_id uuid,
   time timestamp,
   heart_rate int,
   PRIMARY KEY (pet_chip_id, time)
);`{{execute}}

And insert some data:

`INSERT INTO heartrate_v2(pet_chip_id, time, heart_rate) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-03-04 07:01:05', 100); 
INSERT INTO heartrate_v2(pet_chip_id, time, heart_rate) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-03-04 07:01:10', 90); 
INSERT INTO heartrate_v2(pet_chip_id, time, heart_rate) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-03-04 07:01:50', 96); 
INSERT INTO heartrate_v2(pet_chip_id, time, heart_rate) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-04-04 07:01:50', 99);`{{execute}} 

In such a case, the first part of the Primary Key is called the Partition Key (pet_chip_id in the above example), and the second part is called the Clustering Key (time).

A Primary Key is composed of 2 parts:

* The Partition Key is responsible for data distribution across the nodes. It determines which node will store a given row. It can be one or more columns.

![](https://university.scylladb.com/topic/primary-key-partition-key-clustering-key/architecture_ring_modified/#main)

* The Clustering Key is responsible for sorting the rows within the partition. It can be zero or more columns.


Now we can query according to the time:
`SELECT * from heartrate_v2 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23 AND time >='2019-03-04 07:01:00' AND time <='2019-03-04 07:02:00';`{{execute}}

This also solves the problem we saw with the table heartrate_v1. Now each pet can have a different heart rate written for each period. This makes more sense and goes back to the first step in data modeling, thinking about which queries will be performed and taking that into account when creating the data model.

Read the data for the same pet, to see that it wasn't overwritten this time:

`SELECT * from heartrate_v2 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23;`{{execute}} 

both the Partition Key and the Clustering Key can include more than one column, so we can define the following table 

CREATE TABLE heartrate_v3 (
   pet_chip_id uuid,
   time timestamp,
   heart_rate int,
   pet_name text,
   PRIMARY KEY ((pet_chip_id, time), pet_name)
);

And insert some data:
INSERT INTO heartrate_v3(pet_chip_id, time, heart_rate, pet_name) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-03-04 07:01:10', 90, 'Duke'); 

In this case, the partition key includes two columns: pet_chip_id and time, and the clustering key is pet_name. Keep in mind that every query must include all columns defined in the partition key. So, if we try this query:

SELECT * from heartrate_v3 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23;

It fails. Let’s try again with the entire partition key. This time it succeeds. 
SELECT * from heartrate_v3 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23 AND time ='2019-03-04 07:01:10' AND pet_name = 'Duke'; 

Similarly, if we want each partition to be based on the pet_chip_id but to be able to query according to pet_name and heart_rate:
CREATE TABLE heartrate_v4 (
   pet_chip_id uuid,
   time timestamp,
   heart_rate int,
   pet_name text,
   PRIMARY KEY (pet_chip_id, pet_name, heart_rate)
);

INSERT INTO heartrate_v4(pet_chip_id, time, heart_rate, pet_name) VALUES (123e4567-e89b-12d3-a456-426655440b23, '2019-03-04 07:01:10', 90, 'Duke'); 

SELECT * from heartrate_v4 WHERE pet_chip_id = 123e4567-e89b-12d3-a456-426655440b23 AND pet_name = 'Duke' AND heart_rate = 90;

We insert some data and query according to the partition key and clustering key, in the correct order, so it works. 

