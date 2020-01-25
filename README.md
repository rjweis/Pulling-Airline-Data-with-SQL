# SQL Queries
SQL Example Queries  
Data can be found [here.](https://www.kaggle.com/usdot/flight-delays#flights.csv)  

We can use `PRAGMA table_info(table_name)` to see the columns available in each table. For example,  
```SQL
PRAGMA table_info(airlines);
PRAGMA table_info(airports);
PRAGMA table_info(flights);
```  
will give us the following:  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/airlines_columns.PNG">
</p>  

<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/airports_columns.PNG">
</p>  

<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/flights_columns.PNG">
</p>  


**1. Create a report that lists: ORIGIN_AIRPORT, ORIGIN AIRPORT NAME, DESTINATION_AIRPORT, DESTINATION AIRPORT NAME, AND DISTANCE.**  
```SQL
select distinct origin_airport, 
    (select airport
    from airports as a
    where a.acode = f.origin_airport) as origin_airport_name,
    destination_airport, 
    (select airport
    from airports as a
    where a.acode = f.destination_airport) as destination_airport_name,
    distance
from airports as a, flights as f;
```  
<p align="center">
  <img width="860" height="200" src="https://github.com/rjweis/sql-queries/blob/master/q1.PNG">
</p>  

**2. Provide the query that allows you to answer the following question:  Which airport has the most departing flights?**  
```SQL
select origin_airport, count(destination_airport) as number_of_departing_flights
from delays.flights
group by origin_airport
order by number_of_departing_flights desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q2.PNG">
</p>  

**3. Create a report that lists all one-stop flights from Boston (BOS) to San Francisco (SFO). Limit the report to those flights that departed on MONTH=1, DAY=1, and DAY_OF_WEEK=4.**  
```SQL
select first_leg.*, second_leg.*, scheduled_departure_from_layover-scheduled_arrival_at_layover as lay_over_wait
from (select 
        flight_number as first_flight_number, 
        origin_airport, 
        scheduled_arrival as scheduled_arrival_at_layover, 
        destination_airport as lay_over_one
        from delays.flights as f
        where f.origin_airport = "BOS" and f.destination_airport != "SFO" and
        f.month = 1 and
        f.day = 1 and 
        f.day_of_week= 4
        ) as first_leg,
        
    (select 
        origin_airport as lay_over_two, 
        destination_airport, 
        scheduled_departure as scheduled_departure_from_layover, 
        flight_number as second_flight_number
        from delays.flights as f
        where f.origin_airport != "BOS" and f.destination_airport = "SFO" and
        f.month = 1 and
        f.day = 1 and 
        f.day_of_week= 4) as second_leg
        
where lay_over_one = lay_over_two and 
    lay_over_wait > 60
order by lay_over_wait desc; 
```  
![q3 image](https://github.com/rjweis/sql-queries/blob/master/q3.PNG)

**4. Provide the query that allows you to answer the following question:  Which non-stop route has the most cancelled flights in 2015? (CANCELLED -- Flight Cancelled (1 = cancelled))**  
```SQL
select origin_airport, destination_airport, count(cancelled) as number_of_cancelled_flights
from delays.flights
where cancelled = 1
group by origin_airport, destination_airport
order by number_of_cancelled_flights desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q4.PNG">
</p>  

**5. Provide the query that allows you to answer the following question:  Which airlines have the most cancelled flights due to Airline/Carrier cancellation reasons? – Your report should list: AIRLINE CODE, AIRLINE NAME, and the count of flights cancelled due to the specified reasons. Order your report by descending count. (CANCELLATION_REASON: A - Airline/Carrier; B - Weather; C - National Air System; D - Security)**  
```SQL
select airline, count(cancellation_reason) as number_of_airline_cancellations
from delays.flights
where cancellation_reason = 'A'
group by airline
order by number_of_airline_cancellations desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q5.PNG">
</p>  

**6. Provide the query that allows you to answer the following question: Which day of the week has the longest average taxi out. (day_of_week: 1--7, 1 = Monday, 2 = Tuesday, etc)**  
```SQL
select avg(taxi_out), day_of_week
from delays.flights
group by day_of_week
order by avg(taxi_out) desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q6.PNG">
</p>  

**7. Compute the total departure delay of each airline across all flights. Some departure delays may be negative (indicating an early departure); they should reduce the total, so you don't need to handle them specially. Name the output columns AIRLINE and DELAY. Order the report by ascending airline.**  
```SQL
select airline as AIRLINE, sum(departure_delay) as DELAY
from delays.flights
group by airline
order by airline asc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q7.PNG">
</p>  

**8. Find the names of all airlines that ever had more than 100 cancellations in one day (i.e., a specific day/month). Return only the names of the airlines (not airline code). Do not return any duplicates (i.e., airlines with the exact same name).**   
```SQL
select distinct a.airline
from delays.airlines as a, delays.flights as f 
where cancelled = 1
    and a.code = f.airline
group by a.airline, month, day
having count(cancelled) > 100;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q8.PNG">
</p>  

