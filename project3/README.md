## Project 3: Advanced SQL Assignment, CMSC424, Fall 2016

*The assignment is to be done by yourself.*

### Setup

As before we have created a VagrantFile for you. The main differences here are that we have loaded a skewed database (`flightsskewed`), and also installed Java. Start by doing `vagrant up` and `vagrant ssh` as usual. 

### Assignment Questions

**Question 1 (.5 pt)**: Consider the following query which finds the number of flights taken by users whose name starts with 'William'. Before doing this, make sure to create a new database with the provided `large-skewed.sql` file, and use that database for all the questions in this assignment. The `VagrantFile` we have provided already does this.

```
select c.customerid, c.name, count(*)
from customers c join flewon f on (c.customerid = f.customerid and c.name like 'William%') 
group by c.customerid, c.name 
order by c.customerid;
```

The result however does not contain the users whose name contains 'William' but who have no status update (e.g., `cust731`). So we may consider
modifying this query to use a left outer join instead, so we get those users as well: 

```
select c.customerid, c.name, count(*)
from customers c left outer join flewon f on (c.customerid = f.customerid and c.name like 'William%') 
group by c.customerid, c.name 
order by c.customerid;
```

Briefly explain why this query does not return the expected answer (as below), and rewrite the query so that it does. 

The final answer should look like this:
```
	customerid |              name              | count 
	------------+--------------------------------+-------
	cust727    | William Harris                 |     4
	cust728    | William Hill                   |     6
	cust729    | William Jackson                |     6
	cust730    | William Johnson                |     5
	cust731    | William Lee                    |     0
	cust732    | William Lopez                  |     6
	cust733    | William Martinez               |     0
	cust734    | William Mitchell               |     6
	cust735    | William Moore                  |     5
	cust736    | William Parker                 |     4
	cust737    | William Roberts                |     8
	cust738    | William Robinson               |     7
	cust739    | William Rodriguez              |     5
	cust740    | William Wright                 |     8
	cust741    | William Young                  |     5
	(15 rows)
```

---
**Question 2 (.5 pt)**: EXPLAIN can be used to see the query plan used by the database system to execute a query. For the following
query, draw the query plan for the query, clearly showing the different operators and the options they take. The query corresponds to question 3 in Project 1.
```
select a.city, source, b.city, dest, count(*) number_of_flights
from flights, airports a, airports b
where flights.source = a.airportid and flights.dest = b.airportid
group by a.city, source, b.city, dest
having count(*) > 1
order by number_of_flights desc, a.city asc, b.city asc;
```

---
**Question 3 (.5 pt)**: Similarly draw the query plan for the following query, and annotate which operators are responsible for creating `temp1`, `temp2`, and the final answer.

```
with temp1 as (
        select customerid, airlineid, count(*) as num_flights
        from flewon natural join flights
        group by customerid, airlineid
),
temp2 as (
        select customerid
        from temp1 t1
        where num_flights = (select max(num_flights) from temp1 t2 where t1.customerid = t2.customerid)
            and airlineid = 'SW'
)
select customers.customerid, name
from customers, temp2
where customers.customerid = temp2.customerid and customers.frequentflieron != 'SW';
```

---
**Question 4 (.5 pt)**: The EXPLAIN output also shows how many tuples the optimizer expects to be generated after each operation (`rows`). EXPLAIN ANALYZE 
executes the query and also shows the **actual** number of tuples generated when the query plan was executed. 

For the following two queries, identify some of the key intermediate results that were generated by the database, what were their estimated cardinalities (sizes), and what were their actual cardinalities, using EXPLAIN ANALYZE. Did the database generally do a good job on estimating the sizes of the intermediate results? Can you trace the differences in the actual cardinalities for the two queries (which only differ in one constant) to the properties of the data?
```
Query 1:
select c.name, count(*) 
from customers c, flights fl, flewon fo 
where extract(month from c.birthdate) = 1 and c.customerid = fo.customerid and fl.flightid = fo.flightid
      and fl.airlineid = 'AA'
group by c.name;
```
```
Query 2:
select c.name, count(*) 
from customers c, flights fl, flewon fo 
where extract(month from c.birthdate) = 2 and c.customerid = fo.customerid and fl.flightid = fo.flightid
      and fl.airlineid = 'AA'
group by c.name;
```

---
**Question 5 (1 pt)**: [Trigger] 
Let's create a table `NumberOfFlightsTaken(customerid, customername, numflights)` to keep track of the total number of flights taken by each customer. Since this is a derived table (and not a view), it will not be kept up-to-date by the database system.  Use the following command for doing this:
```
create table NumberOfFlightsTaken as 
select c.customerid, c.name as customername, count(*) as numflights 
from customers c join flewon fo on c.customerid = fo.customerid
group by c.customerid, c.name;
```

Write a `trigger` to keep this new table updated when a new entry is inserted into or a row is deleted from the `flewon` table. Remember the customerid corresponding to the new flewon update may not exist in the `NumberOfFlightsTaken` table at that time and it should be added to the table with a count of 1, in that case. Similarly, if a deletion of a row in `flewon` results in a user not having any flights, then the corresponding tuple for that user in `NumberOfFlightsTaken` should be deleted.

The trigger code should be submitted in `trigger.sql` file, as straight SQL. Running `psql -f trigger.sql flightsskewed` should generate the trigger without errors.

---
**Question 6 (2 pt)**:  One of more prominent ways to use a database system is using an external client, using APIs such as ODBC and JDBC.
This allows you to run queries against the database and access the results from within say a Java program.

Here are some useful links:
- [Wikipedia Article](http://en.wikipedia.org/wiki/Java_Database_Connectivity)
- [Another resource](http://www.mkyong.com/java/how-do-connect-to-postgresql-with-jdbc-driver-java/)
- [PostgreSQL JDBC](http://jdbc.postgresql.org/index.html)

The last link has detailed examples in the `documentation` section. The `project3` directory (in the git repository) also contains an example 
file (`JDBCExample.java`). To run the JDBCExample.java file, do:
`javac JDBCExample.java` followed by `java -classpath .:./postgresql-9.0-801.jdbc3.jar JDBCExample`.

Your task to write a JDBC program that will take in JSON updates and insert appropriate data into the database. 
Two types of updates should be supported:
- New customer, where information about a customer is provided, in the following format. You can assume that the frequent flier airline name matches exactly what is there in the `airlines' table.
```
{ "newcustomer": 
	{
	"customerid": "cust1000",
	"name": "XYZ",
	"birthdate": "1991-12-06",
	"frequentflieron": "Southwest Airlines"
	}
}
```
- Flight information, where information about the passengers in a flight is provided. 
```
{ "flightinfo": {
	"flightid": "DL119",
	"flightdate": "2015-09-25",
	"customers": [
		{"customer_id": "cust94"}, 
		{"customer_id": "cust102"},
		{"customer_id": "cust1000", "name": "XYZ", "birthdate": "1991-12-06", "frequentflieron": "DL"}
		]
	}
}
```
Note that, in some cases, the customer_id provided may not be present in the database (the last one above). In that case, you have to first update the `customers` table, before adding tuples to the `flewon` table.

Your code should also catch one type of error: If the `customerid` for a `newcustomer` update is already present or if the `frequentflieron` does not have a match in the airlines table, it should print out an error. You don't need to handle errors for the second update type.

### Parsing JSON
There are quite a few Java JSON parsing libraries out there to simplify the parsing process. To simplify matters, we have provided a JSON parsing library for you to use: `json-simple-1.1.1.jar`. Here is the webpage for the library: https://github.com/fangyidong/json-simple. Some code samples can be found in `src/test` directory there, or you can also see some examples here: https://www.mkyong.com/java/json-simple-example-read-and-write-json/

The provided `JSONProcessing.java` file already takes care of the input part, and you just need to focus on finishing the `processJSON(String json)` function. An example JSON file is provided: `example.json`.

### Submission Instructions
We have provided a `answers.docx` file -- fill in your answers to the first 4 questions into that doc file (including scanned PDFs or images).
You may want to use some tool (e.g., Google Drawing) to draw query plans.
In addition, submit your `trigger.sql` and `JSONProcessing.java` files separately.

We will provide some additional test scripts later.