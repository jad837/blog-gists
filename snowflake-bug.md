# Debug Story - How I found a Bug in JDBC driver.

If you’ve been working as Software Engineer for long you always face some bugs & most often than not you have those ‘Eureka’ moments which solve the problem. I have been using JDBC drivers for past 3 years and never have I ever thought that I would find a bug within them.

## **The Scenario:**

This year in 2023, I've been involved in migrating our tech stack from Java 8 to Java 17, updating various dependencies such as Spring Boot, JDBC drivers, and Snowflake JDBC drivers. In November, I stumbled upon a peculiar bug in an API we had migrated. The API, utilizing the Snowflake data warehouse, fetched `Date` type data and returned it as a response. This API employed [JOOQ](https://www.jooq.org/), the Snowflake JDBC driver, and the JDBC driver under the hood, with the application built using Spring Boot.

## The Issue:

The problem arose when using a gRPC endpoint to fetch the maximum date from multiple tables. The dates were stored as either `Date` or `String` data types, depending on the queried table. 

Let’s consider the following 2 tables as examples.

**Table 1 is named** **PURCHASES having purchased_on column with type as `Date`.**

| purchased_on | quantity |
| --- | --- |
| 2023-01-01 | 10 |
| 2022-12-31 | 32 |

**Table 2 is named** **REGISTRATIONS having registered_on column with type as `String`**

| registered_on | model_no |
| --- | --- |
| 2023-01-01 | ABCQ1 |
| 2022-12-31 | XYZP2 |

Consider the following expected and actual responses:

### Expected Response:

```java
// when max date is checked for PURCHASES TABLE
{
	"max_date": '2023-01-01'
}

// when max date is checked for REGISTRATIONS TABLE
{
	"max_date": '2023-01-01'
}
```

However, we observed an unexpected response, particularly in the stage and prod environments. We expected the max_date to be 2023-01-01 as this is the maximum date if we consider both above example tables. But, we were getting `expected date -1` when API was called for fetching max date from PURCHASES table where column type is `Date`.

### Actual response:

```java
// when max date is checked for PURCHASES TABLE
{
	"max_date": '2022-12-31'
}

// when max date is checked for REGISTRATIONS TABLE
{
	"max_date": '2023-01-01'
}
```

The initial suspicion was a timezone issue since we were working in the India/Kolkata timezone, while our servers were in the Americas/NewYork timezone. Now, if this was an issue on our side of the code it should’ve been consistent throughout the code-base & should happen for both the tables regardless of the date field data type.

### Code for fetching data from database:

I created sample code with JOOQ to fetch data from database.

```java
public LocalDate getDataFromJooqQuery(Connection connection, List<String> dates, TableFields tableFields ) {
    var context = DSL.using(connection);
    var maxDateField = DSL.field(tableFields.fieldName).as("max_date");
    var registerdDateField = DSL.field(tableFields.fieldName);

    var select = context.select(maxDateField).from(tableFields.name()).where(registerdDateField.in(dates)).limit(1);
    var maxdate = context.fetchOne(select).into(LocalDate[].class);

    return maxdate[0];
}
```

So, our transformation for columns `String` to `LocalDate` & `Date` to `LocalDate` was handled by [JOOQ](https://www.jooq.org/)  library as we were using it for dynamic query generation. 

To run the application I was using following command:

```java
java -jar myapplication-0.0.1-SNAPSHOT.jar -Duser.timezone=Americas/NewYork
```

### Potential hypothesis having issues:

1. `dates`  we were sending wrong Dates while querying. I was able to log dates & see the data getting passed as dates. As the dates was expected on server as well so further investigation into this wasn't needed.
2. JOOQ not able to accurately convert database  `Date` type to `LocalDate.class` .

### The Breakthrough:

I was able to confirm that below line was having `maxDate` which was `expected date- 1` which meant that we were having an issue fetching the data into LocalDate. 

```java
LocalDate[] maxDate = context.fetchOne(select).into(LocalDate[].class);
```

Then I quickly switch from JOOQ to simple JDBC query and hit the same API endpoint. 

Code to fetch data from snowflake using simple string SQL query:

```java
public LocalDate getData(@RequestParam List<String> dates, String tableName) throws SQLException {
    var log = LoggerFactory.getLogger(Example.class);
    log.debug("Fetching max date for table: {} & dates : {}", tableName, dates.toString());
    try (var con = dbConnection()) {
        var statement = con.prepareStatement("SELECT max(purchased_on) AS max_date FROM PURCHASES where purchased_on IN ('2023-01-01', '2022-12-31) limit 1");
        var result = statement.executeQuery();
        result.next();
        var maxDate = result.getDate("max_date");

        var jooqMaxDate = getDataFromJooqQuery(con, dates, TableFields.valueOf(tableName));
        log.debug("jooqmaxdate : {}", jooqMaxDate.toString());
        log.debug("queryMaxDate : {}", maxDate.toString());

        return LocalDate.ofInstant(maxDate.toInstant(), ZoneId.of("America/New_York"));
    } catch (Exception e) {
        return null;
    }
```

Switching from JOOQ to a simple JDBC query produced the same unexpected result, leading me to suspect a JDBC connection parameter.

### **Uncovering the JDBC Connection Parameter Issue:**

Further investigation revealed that Snowflake had migrated its default result format to Apache Arrow. Our application, using JOOQ to fetch data, couldn't seamlessly translate the data from Arrow to the desired Java classes. To maintain compatibility with JOOQ, we instructed the Snowflake driver to convert the `ResultSet` into JSON format. 

We were using following connection parameter to tell snowflake driver to use JSON format for results.

```java
ALTER SESSION JDBC_QUERY_RESULT_FORMAT='JSON'
```

Removing this connection parameter resolved the issue. Then I went over to Snowflake JDBC repository on GitHub & was able confirm that this as a known problem ([link](https://github.com/snowflakedb/snowflake-jdbc/issues/1519)). 

### The Fix:

I had Two solutions:

1. **Disable TimeZone Argument:**

Add the following connection parameter:

```java
ALTER SESSION JDBC_FORMAT_DATE_WITH_TIMEZONE=FALSE
```

1. **Switch to ARROW Format**

Improves performance and resolve the issue by switching to the ARROW format.

### Concluding…

This shows that using well known, highly used external libraries serving core functionality can have issues as modern software is subject to evolve and change. Having the necessary depth to know and willingness to debug their code to mark issues always comes in handy. 

Always choose external dependencies with care and don’t be afraid of using that debugger to dive into other’s code

**Happy Coding!**

### References:

1. https://github.com/jad837/snowflake-date-issue
2. https://github.com/snowflakedb/snowflake-jdbc/issues/1519
3. [https://medium.com/snowflake/arrow-database-connectivity-adbc-support-for-snowflake-7bfb3a2d9074](https://medium.com/snowflake/arrow-database-connectivity-adbc-support-for-snowflake-7bfb3a2d9074)
