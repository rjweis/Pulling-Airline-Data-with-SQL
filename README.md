# Pulling Airline Data with SQLite    
![airport](https://thumbor.forbes.com/thumbor/960x0/https%3A%2F%2Fspecials-images.forbesimg.com%2Fdam%2Fimageserve%2F1149089169%2F960x0.jpg%3Ffit%3Dscale)  
*Image Credit: Forbes/Getty*
### Introduction 
Extracting, transforming, and loading data is a vital skill for any analytics professional. In this markdown, we'll take a look at some queries I completed for an assignment at the [Institute for Advanced Analytics](https://analytics.ncsu.edu/) to develop this skill. 

This assignment is about extracting information from a database to answer questions one would have about airline travel. The database contains three tables -- airlines, airports, and flights -- and they can be found [here.](https://www.kaggle.com/usdot/flight-delays#flights.csv)  

First, lets take a look at the columns contained in these tables. We can use `PRAGMA table_info(table_name)` to accomplish this. For example,  
```SQL
PRAGMA table_info(airlines);
PRAGMA table_info(airports);
PRAGMA table_info(flights);
```  
will give us the following:  

<p align="center"><strong>airlines</strong></p>
  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/airlines_columns.PNG">  
</p>  
  
<p align="center"><strong>airports</strong></p>  
  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/airports_columns.PNG">
</p>  
  
<p align="center"><strong>flights</strong></p>  
  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/flights_columns.PNG">
</p>  
  
  
We can see that there is not there is not a single ID column that exists in all the tables. However, we do see that `AIRLINE` is contained in both the airlines table and the flights table. Also, it looks like either `AIRPORT` or `ACODE` from the airports table will join to the flights table on `ORIGIN_AIRPORT` and `DESTINATION_AIRPORT`. We can run a simple query to view these columns and see which one we'll need. 
```SQL
SELECT DISTINCT a.airport, a.acode, f.origin_airport, f.destination_airport
FROM delays.airports as a, delays.flights as f
LIMIT 5;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q1_prep.PNG">
</p>  

It turns out that `ACODE` is the column we'll be using to join the airports and flights tables. So, now we have everything we need to put all three of these tables together! 
   
### Tasks  
**1. Create a report that lists: ORIGIN_AIRPORT, ORIGIN AIRPORT NAME, DESTINATION_AIRPORT, DESTINATION AIRPORT NAME, AND DISTANCE.**  

For this query, we need to get data from both the airports and flights tables. `origin_airport` and `destination_airport` are the airport codes from the flights table, so to get their respective names from the airports table we can use subqueries. `where a.acode = f.origin_airport` and `where a.acode = f.destination_airport` will help us get this information. 

```SQL
SELECT DISTINCT origin_airport, 
    (SELECT airport
    FROM airports AS a
    WHERE a.acode = f.origin_airport) AS origin_airport_name,
    destination_airport, 
    (SELECT airport
    FROM airports AS a
    WHERE a.acode = f.destination_airport) AS destination_airport_name,
    distance
FROM airports AS a, flights AS f;
```  
Now, we know the distance between these airports:  

<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q1.PNG">
</p>  

**2. Provide the query that allows you to answer the following question:  Which airport has the most departing flights?**  

To answer this question, we need to determine if we'll use `origin_airport` or `destination_airport` for our query. We know that for each airport, we want the count of all departing flights. In other words, we want the destination airports, which we can obtain with `count(destination_airport)`. To get the count for *each* airport, we can use `group by origin_airport`. Then, to see which airport has the most, we can simply order our query in descending order with `desc` and use the first row to answer this question.  

```SQL
SELECT origin_airport, count(destination_airport) AS number_of_departing_flights
FROM delays.flights
GROUP BY origin_airport
ORDER BY number_of_departing_flights desc;
```  
  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q2.PNG">
</p>  

Surprisingly, the airport with the most departures is ATL in Atlanta, Georgia! I personally thought it would have been JFK in New York, given the number of incoming and outgoing international flights. A quick Google search does, however, confirm that our data isn't lying to us. ATL was *the busiest airport in the world* in 2018!   
  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/atl.PNG">
</p> 
  
**3. Create a report that lists all one-stop flights from Boston (BOS) to San Francisco (SFO). Limit the report to those flights that departed on MONTH=1, DAY=1, and DAY_OF_WEEK=4.**  

Out of all the questions for this assignment, this one is by far the trickiest. Why? Because we need *one-stop flights*, but our data only has *direct-flights*.  
  
However, we can use the fact that the returned records need to begin at BOS and end at SFO to help us think through this problem. Essentially, our query will have two `origin_airport` columns and two `destination_airport` columns. Let's think about it like this:  

<p align="center"><strong>origin_airport1 --> destination_airport1 --> origin_airport2 --> destination_airport2</strong></p>  
  
Filling in our origin and destination airport constraints, we have:  

<p align="center"><strong>BOS --> destination_airport1 --> origin_airport2 --> SFO</strong></p>  
  
Further, we know that:  

<p align="center"><strong>destination_airport1 != BOS</strong></p>  

<p align="center"><strong>origin_airport2 != SFO</strong></p>   

<p align="center"><strong>destination_airport1 = origin_airport2</strong></p>  

This is because 1) if you are flying from Boston, you are necessarily going somewhere that is *not* Boston; 2) if you are taking a one-stop flight to San Francisco, by definition your stop *can't* be San Francisco; and 3) the destination of your first flight *must* be the origin of your second flight.  

Also, we must keep in mind that we need ample time at our layover to switch flights and to account for any potential delays on our first flight. Lets give ourselves at least 60 minutes. In other words,  

<p align="center"><strong>scheduled_departure_from_origin_airport2 - scheduled_arrival_at_destination_airport1 > 60</strong></p>
  
Now that we have figured out the logic, let's write the query. I find it helpful to break down the flight into a `first_leg` and `second_leg`, which is what I named the first and second subqueries respectively. 

```SQL
SELECT first_leg.*, second_leg.*, scheduled_departure_from_layover-scheduled_arrival_at_layover AS lay_over_wait
FROM (SELECT 
        flight_number AS first_flight_number, 
        origin_airport, 
        scheduled_arrival AS scheduled_arrival_at_layover, 
        destination_airport AS lay_over_one
        FROM delays.flights AS f
        WHERE f.origin_airport = "BOS" AND f.destination_airport != "SFO" AND
        f.month = 1 AND
        f.day = 1 AND 
        f.day_of_week = 4
        ) AS first_leg,
        
    (SELECT 
        origin_airport AS lay_over_two, 
        destination_airport, 
        scheduled_departure AS scheduled_departure_from_layover, 
        flight_number AS second_flight_number
        FROM delays.flights AS f
        WHERE f.origin_airport != "BOS" AND f.destination_airport = "SFO" AND
        f.month = 1 AND
        f.day = 1 AND 
        f.day_of_week = 4
        ) AS second_leg
        
WHERE lay_over_one = lay_over_two AND 
    lay_over_wait > 60
ORDER BY lay_over_wait DESC; 
```  
![q3 image](https://github.com/rjweis/sql-queries/blob/master/q3.PNG)
 
We can use `flight_number` for both legs to visually verify that each row is unique. For instance, in rows 1 and 2 the first leg is identical, but the second leg is different.    

But, if we look closer, we can see that something is wrong in the `lay_over_wait` column. The wait should actually be the amount of time between 12:39 a.m. and 6:30 p.m., which would be about 18 hours. However, these columns were not processed as time variables (as shown in our first `PRAGMA table_info()`), so the wait was actually computed as 1830 - 39, which gives us our result of 1,791 minutes. This would be nearly 30 hours, which is a *very* long wait and violates our where clause `f.day = 1` for both legs of the trip.  

After searching all over [sqlite.org](https://www.sqlite.org/) and [Stack Overflow](https://stackoverflow.com/), it seems that fixing this type of issue isn't a strong suit of SQLite. The best solution I've been able to come up with is to duplicate the flights table, insert a colon in the `scheduled_departure` and `scheduled_arrival` columns so that they can be processed as time variables, and then use `julianday()` to calculate the `lay_over_wait_time`. Going into further detail is beyond the scope of this markdown, but feel free to share any other solutions to solve this problem a little more elegantly.   

```SQL
CREATE table delays.flights_v1 AS
SELECT *, 
    CAST(
        SUBSTR(scheduled_departure, 1, 2) || ':' || SUBSTR(scheduled_departure, 3, 2) AS TEXT) AS new_scheduled_arrival,
    CAST(
        SUBSTR(scheduled_arrival, 1, 2) || ':' || SUBSTR(scheduled_arrival, 3, 2) AS TEXT) AS new_scheduled_departure
FROM delays.flights;

SELECT first_leg.*, second_leg.*, CAST((
    JULIANDAY(scheduled_departure_from_layover) - JULIANDAY(scheduled_arrival_at_layover)
    ) * 24 * 60 AS intgeger) AS lay_over_wait
FROM (SELECT 
        flight_number AS first_flight_number, 
        origin_airport, 
        scheduled_arrival AS scheduled_arrival_at_layover, 
        destination_airport AS lay_over_one
        FROM delays.flights AS f
        WHERE f.origin_airport = "BOS" AND f.destination_airport != "SFO" AND
        f.month = 1 AND
        f.day = 1 AND 
        f.day_of_week = 4
        ) AS first_leg,
        
    (SELECT 
        origin_airport AS lay_over_two, 
        destination_airport, 
        scheduled_departure AS scheduled_departure_from_layover, 
        flight_number AS second_flight_number
        FROM delays.flights AS f
        WHERE f.origin_airport != "BOS" AND f.destination_airport = "SFO" AND
        f.month = 1 AND
        f.day = 1 AND 
        f.day_of_week = 4
        ) AS second_leg
        
WHERE lay_over_one = lay_over_two AND 
    lay_over_wait > 60
ORDER BY lay_over_wait DESC; 
```  
This finally gives us the result we were looking for:  

![q3 image](https://github.com/rjweis/sql-queries/blob/master/q3v1.PNG)  

**4. Provide the query that allows you to answer the following question:  Which non-stop route has the most cancelled flights in 2015?**  

The main idea of this question is to use *both* the `origin_airport` and `destination_airport` columns in our `groupby` clause. This means that each unique pairing of `origin_airport` to `destination_airport` will serve as our 'group' (i.e., the non-stop routes we're looking for). As long as we remember to filter the data for only the cancelled flights (i.e., `where cancelled = 1`), we have everything we need to write this query.   

```SQL
SELECT origin_airport, destination_airport, COUNT(cancelled) AS number_of_cancelled_flights
FROM delays.flights
WHERE cancelled = 1
GROUP BY origin_airport, destination_airport
ORDER BY number_of_cancelled_flights desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q4.PNG">
</p>  

**5. Provide the query that allows you to answer the following question:  Which airlines have the most cancelled flights due to Airline/Carrier cancellation reasons? – Your report should list: AIRLINE CODE, AIRLINE NAME, and the count of flights cancelled due to the specified reasons. Order your report by descending count. (CANCELLATION_REASON: A - Airline/Carrier; B - Weather; C - National Air System; D - Security)** 
  
The same logic from the previous question applies here. Since we're wanting the amount of cancelled flights for each airline, we need to use `group by`. We'll also filter the data for `where cancellation_reason = 'A'`.

```SQL
SELECT airline, count(cancellation_reason) AS number_of_airline_cancellations
FROM delays.flights
WHERE cancellation_reason = 'A'
GROUP BY airline
ORDER BY number_of_airline_cancellations desc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q5.PNG">
</p>  

**6. Provide the query that allows you to answer the following question: Which day of the week has the longest average taxi out. (day_of_week: 1--7, 1 = Monday, 2 = Tuesday, etc)**  

To calculate the longest average taxi out time, we can simply use `avg(taxi_out)`. Since we want the average taxi out time for each day, we will need to use `group by day_of_week`.  

```SQL
SELECT AVG(taxi_out), day_of_week
FROM delays.flights
GROUP BY day_of_week
ORDER BY avg(taxi_out) DESC;
```  

<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q6.PNG">
</p>  
  
It's interesting that Saturday and Sunday have the lowest average taxi out times. It would be interesting to see if these times correlate with the number of passengers travelling per day, airline, travel route, etc.  

**7. Compute the total departure delay of each airline across all flights. Some departure delays may be negative (indicating an early departure); they should reduce the total, so you don't need to handle them specially. Name the output columns AIRLINE and DELAY. Order the report by ascending airline.**  

Similarly to the above query, we'll now use the `sum()` function to get the total delay times. To compute these totals for each airline, we'll use `group by airline`.
```SQL
SELECT airline as AIRLINE, sum(departure_delay) as DELAY
FROM delays.flights
GROUP BY airline
ORDER BY airline asc;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q7.PNG">
</p>  

**8. Find the names of all airlines that ever had more than 100 cancellations in one day (i.e., a specific day/month). Return only the names of the airlines (not airline code). Do not return any duplicates (i.e., airlines with the exact same name).**  

For this query, we need to ensure that we join the `airlines` and `flights` tables properly by using `airlines.code = flights.airline`. Further, after grouping by each unique pairing of `airline`, `month`, and `day`, we need to filter out these groups for airlines that have more than 100 cancelled flights. This can be accomplished with `having count(cancelled) > 100`.  

```SQL
SELECT distinct a.airline
FROM delays.airlines AS a, delays.flights AS f 
WHERE cancelled = 1
    AND a.code = f.airline
GROUP BY a.airline, month, day
HAVING COUNT(cancelled) > 100;
```  
<p align="center">
  <img src="https://github.com/rjweis/sql-queries/blob/master/q8.PNG">
</p>  

