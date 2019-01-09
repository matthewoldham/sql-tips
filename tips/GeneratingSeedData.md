Generating Random (Seed) Data
===============

# Background

Many times one needs to seed their database with dummy data or randomize/deidentify existing data for testing purposes. It's pretty amazing what you can accomplish out of the box with SQL. Following are some common scenarios.

## Basic random value

```
postgres# SELECT RANDOM();

      random       
-------------------
 0.249834946822375
(1 row)
```

According to the [PostgreSQL documentation](https://www.postgresql.org/docs/current/functions-math.html) the `random` function returns a `random value in the range 0.0 <= x < 1.0`.

Building on this simple function, can we generate a random value in a known range?

## Random value in a given range
```
postgres# WITH base AS (
   SELECT 1 AS startingValue,
          100 AS endingValue
)
SELECT (startingValue + (endingValue - startingValue) * RANDOM())::NUMERIC
  FROM base;

     numeric      
------------------
 57.9347848887555
(1 row)
```

To make this reusable for other scenarios that will build upon it, let's convert it to a function:

```
postgres# CREATE FUNCTION randomNumber(startingValue IN NUMERIC, endingValue IN NUMERIC)
RETURNS NUMERIC AS
$$
   SELECT (startingValue + (endingValue - startingValue) * RANDOM())::NUMERIC;
$$
LANGUAGE 'sql'
VOLATILE;

CREATE FUNCTION
```

Now, let's test it out:

```
postgres# SELECT randomNumber(1,10),
                 randomNumber(1,100),
                 randomNumber(99,105),
                 randomNumber(99.000,99.999);

   randomnumber   |   randomnumber   |   randomnumber   |   randomnumber   
------------------+------------------+------------------+------------------
 1.96116291591898 | 47.4438195298426 | 103.643512582406 | 99.6020536908642
(1 row)
```

Can we get random integers?

```
postgres# SELECT randomNumber(1,10)::INTEGER,
                 randomNumber(1,100)::INTEGER,
                 randomNumber(99,105)::INTEGER;

 randomnumber | randomnumber | randomnumber 
--------------+--------------+--------------
            3 |           77 |          103
(1 row)
```

Great! Now we have something we can use in multiple scenarios. Let's dive into the fun stuff!

## Random boolean values

Booleans become really easy at this point since PostgreSQL recognizes `0` as `FALSE` and 1 as `TRUE`.

```
postgres# SELECT CAST(ROUND(randomNumber(0,1))::INTEGER AS BOOLEAN);

 round 
-------
 f
(1 row)

postgres# SELECT CAST(ROUND(randomNumber(0,1))::INTEGER AS BOOLEAN);

 round 
-------
 f
(1 row)

postgres# SELECT CAST(ROUND(randomNumber(0,1))::INTEGER AS BOOLEAN);

 round 
-------
 t
(1 row)
```

Let's turn this one into a function as well, which will come in handy very shortly:

```
postgres# CREATE FUNCTION randomBoolean()
RETURNS BOOLEAN AS
$$
   SELECT CAST(ROUND(randomNumber(0,1))::INTEGER AS BOOLEAN);
$$
LANGUAGE 'sql'
VOLATILE;

CREATE FUNCTION
```

Now let's give this a trial run to make sure it's working like we expect:

```
postgres# SELECT randomBoolean(),
                 randomBoolean(),
                 randomBoolean(),
                 randomBoolean();

 randomboolean | randomboolean | randomboolean | randomboolean 
---------------+---------------+---------------+---------------
 f             | t             | t             | f
(1 row)
```

Half TRUE and half FALSE - perfect!

## Weighted random boolean

Many times you need to simulate authentic variation in your seed data, and weighting is one way to accomplish this. For example, I may want to generate boolean values for a population of data where the majority of values should be false. We're going to accomplish this with yet another function so that we can easily reference it in SQL:

```
postgres# CREATE FUNCTION randomWeightedBoolean(trueWeight IN NUMERIC)
RETURNS BOOLEAN AS
$$ 
   SELECT randomNumber(0,1) < trueWeight
$$
LANGUAGE 'sql' 
VOLATILE;

CREATE FUNCTION
```

Now, let's use my hypothetical above. If I want the false values to be the majority, I can simply pass in a lower `trueWeight`. Since our base `randomNumber` function returns a value between 0 and 1, I need to represent `trueWeight` as a decimal "percentage". The other part of testing this out is to generate enough values to determine if my weighting is working. For this we'll use the extremely handy PostgreSQL function [generate_series](https://www.postgresql.org/docs/current/functions-srf.html).

```
postgres# WITH base AS (
   SELECT randomWeightedBoolean(.75) AS randomBoolean
     FROM GENERATE_SERIES(1,100)
)
SELECT randomBoolean, COUNT(*) AS "Occurrences"
  FROM base
 GROUP BY randomBoolean;
 
 randomboolean | Occurrences 
---------------+-------------
 f             |          27
 t             |          73
(2 rows)
```

So, across 100 iterations, our weighted boolean function generated approximately 75% `FALSE` values!

## Random row(s) from a table

Another interesting facet of the `random` function is that is can be used in the `ORDER BY` clause. Let's take a look at an example by generating a sample data set (again, using `generate_series`) with and without a randomized ordering.

```
/* Without Randomized Ordering */
postgres# WITH base AS (
   SELECT generate_series(1,10) AS val 
)
SELECT val
  FROM base;

 val 
-----
   1
   2
   3
   4
   5
   6
   7
   8
   9
  10
(10 rows)

/* With Randomized Ordering */
postgres# WITH base AS (
   SELECT generate_series(1,10) AS val 
)
SELECT val
  FROM base
 ORDER BY RANDOM();

 val 
-----
   9
   4
   5
   7
   2
  10
   3
   1
   6
   8
(10 rows)
```

Pretty cool! Now we can easily grab a random single row from any given data set using the `LIMIT` clause:

```
postgres# WITH base AS (
   SELECT generate_series(1,10) AS val 
)
SELECT val
  FROM base
 ORDER BY RANDOM()
 LIMIT 1;

 val 
-----
   7
(1 row)
```

## Random value from an enumerated list

This is one of the most common scenarios I have run across. We need randomized data, but the values should be restricted to an enumerated list that we know in advance. 

For example, say we wanted to return a random value from the list `Cyan, Magenta, Yellow, Black`. The first challenge will be to turn our list into a data set. This is easily done by leveraging POstgreSQL [ARRAYs](https://www.postgresql.org/docs/current/arrays.html). Let's give it a try.

```
postgres# WITH base AS (
   SELECT ARRAY['Cyan','Magenta','Yellow','Black'] AS colors
)
SELECT colors
  FROM base;

           colors            
-----------------------------
 {Cyan,Magenta,Yellow,Black}
(1 row)
```

Our list of values has been converted to an ARRAY. But how do we return them as individual rows? PostgreSQL includes a nifty `UNNEST` function that does this for us:

```
postgres# WITH base AS (
   SELECT ARRAY['Cyan','Magenta','Yellow','Black'] AS colors
)
SELECT UNNEST(colors) AS color
  FROM base;

  color  
---------
 Cyan
 Magenta
 Yellow
 Black
(4 rows)
```

Building on several of our previous examples, we can create another function that allows us to use this approach for any enumerated list. We'll make special use of the `[]` notation for defining a `TEXT` array, and treating all input values as `TEXT` will give us the greatest flexibility.

```
postgres# CREATE FUNCTION randomValueFromList(valueList IN TEXT[])
RETURNS TEXT AS
$$
WITH base AS (
   SELECT val
     FROM UNNEST(valueList) val
)
SELECT val
  FROM base
 ORDER BY RANDOM()
 LIMIT 1
$$
LANGUAGE 'sql'
VOLATILE;

CREATE FUNCTION
```

Now let's test out our new function:

```
postgres# SELECT randomValueFromList(ARRAY['Which','One','Will','It','Be','?']) AS "randomString1",
       randomValueFromList(ARRAY['Yes','No']) AS "randomString2",
       randomValueFromList(ARRAY['Yes','No'])::BOOLEAN AS "randomBoolean1",
       randomValueFromList(ARRAY['True','False'])::BOOLEAN "randomBoolean2",
       randomValueFromList(ARRAY['1','2','3'])::INTEGER "randomInteger",
       randomValueFromList(ARRAY['1.01','1.02','1.03'])::DECIMAL "randomDecimal";

 randomString1 | randomString2 | randomBoolean1 | randomBoolean2 | randomInteger | randomDecimal 
---------------+---------------+----------------+----------------+---------------+---------------
 Will          | No            | f              | f              |             2 |          1.02
(1 row)
```

Wow! I hope you're having as much fun as I am! :sunglasses: :nerd_face:

## Random text

This is an interesting one. We can easily generate a random string. Notice I did not say "meaningful". Here's one way:

```
postgres# SELECT MD5(RANDOM()::TEXT) AS randomString;

           randomstring           
----------------------------------
 fcc80db4edd11b231f72d2f0131af86a
(1 row)
```

We can also shorten it to the desired length (note, using `MD5` will obviously only generate a 32-byte string);

```
postgres# SELECT SUBSTR(MD5(RANDOM()::TEXT),1,10) AS randomString;

 randomstring 
--------------
 aad3e22ff4
(1 row)
```

We can also manipulate the text case:

```
postgres# SELECT UPPER(SUBSTR(MD5(RANDOM()::TEXT),1,10)) AS randomUpperCaseString,
                 INITCAP(SUBSTR(MD5(RANDOM()::TEXT),1,10)) AS randomProperCaseString;

 randomuppercasestring | randompropercasestring 
-----------------------+------------------------
 9E56E1242C            | D26088672b
(1 row)
```

This will only provide limited value, especially if you need longer strings that are more meaningful (e.g. person names, addresses, etc).

To address the length concern, let's start with an example I originally found posted by user `Lyndon S` on [StackOverflow](https://stackoverflow.com/a/47321257). Here is my adaptation, which will produce a random list of 10 characters from the english alphabet:

```
postgres# SELECT SUBSTR('ABCDEFGHIJKLMNOPQRSTUVWXYZ',randomNumber(1,26)::INTEGER,1) AS letter
  FROM GENERATE_SERIES(1,10);

 letter 
--------
 I
 S
 V
 C
 L
 M
 U
 C
 U
 X
(10 rows)
```

Building on that, we can use our old friend `generate_series` along with a new friend `string_agg` to generate random "words" of a specified length (note that I am artifically limited the max length of a string to 100 characters):

```
postgres# WITH base AS (
   SELECT SUBSTR('ABCDEFGHIJKLMNOPQRSTUVWXYZ',randomNumber(1,26)::INTEGER,1) string1,
          SUBSTR('ABCDEFGHIJKLMNOPQRSTUVWXYZ',randomNumber(1,26)::INTEGER,1) string2,
          SUBSTR('ABCDEFGHIJKLMNOPQRSTUVWXYZ',randomNumber(1,26)::INTEGER,1) string3
     FROM GENERATE_SERIES(1,100)
)
SELECT SUBSTR(STRING_AGG(string1,''),1,10) AS stringLength10,
       SUBSTR(STRING_AGG(string2,''),1,5) AS stringLength5,
       SUBSTR(STRING_AGG(string3,''),1,1) AS stringLength1
  FROM base;

 stringlength10 | stringlength5 | stringlength1 
----------------+---------------+---------------
 PRSTUVGWTX     | FVVTY         | B
(1 row)
```

To randomly generate truly meaningful strings requires having a list of meaningful values stored in a table. I have done this before by prepopulating a "dummy names" table from which I can randomly select a value for both the first name and last name of a person, for example.

## Miscellaneous use cases

Using a combination of some of the things we have learned so far (and a few new concepts), here are several interesting use cases I have come across that required randomized seed data:

#### Patient Body Temperature

```
postgres# SELECT ROUND(randomNumber(35.5,38.0),1) AS degreesCelsius;

 degreescelsius 
----------------
           37.1
(1 row)
```

#### Phone Number

```
postgres# SELECT TO_CHAR(ROUND(randomNumber(1111111111,9099999999)),'999-999-9999') AS phone;

     phone     
---------------
  733-442-3908
(1 row)
```

#### Future date in the next 12 months

```
postgres# SELECT CURRENT_DATE AS currentDate,
                 TO_DATE((TO_CHAR(CURRENT_TIMESTAMP,'J')::INTEGER + ROUND(randomNumber(1,365)))::TEXT,'J') AS futureDate;
 
 currentdate | futuredate 
-------------+------------
 2019-01-09  | 2019-11-04
(1 row)
```

#### Random date within the past 30 days

```
postgres# SELECT CURRENT_DATE AS currentDate,
       (CURRENT_DATE-(ROUND(randomNumber(1,29))||' DAYS')::INTERVAL)::DATE AS pastDate;

 currentdate |  pastdate  
-------------+------------
 2019-01-09  | 2018-12-15
(1 row)
```

#### Random date of birth for a person under the age of 18

```
postgres# WITH base AS (
   SELECT CAST(CURRENT_TIMESTAMP - randomNumber(1,18)*365 * INTERVAL '1 day' AS DATE) birthDate
)
SELECT CURRENT_DATE AS currentDate,
       birthDate,
       AGE(CURRENT_DATE,birthDate)
  FROM base;

 currentdate | birthdate  |           age           
-------------+------------+-------------------------
 2019-01-09  | 2006-06-19 | 12 years 6 mons 20 days
(1 row)
```

I'll add more examples as I think of them, but hopefully these demonstrate how much is possible using SQL.

