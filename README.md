## introduction to databases

### navigation

- [background](https://github.com/SOS598-RDM/rdm-databases#background)
- [create the database](https://github.com/SOS598-RDM/rdm-databases#create-the-database)
- [create the tables](https://github.com/SOS598-RDM/rdm-databases#create-the-tables)
- [load data](https://github.com/SOS598-RDM/rdm-databases#load-data-into-the-database)
- [harvesting data](https://github.com/SOS598-RDM/rdm-databases#getting-data-out-of-the-database)
- [assignment](https://github.com/SOS598-RDM/rdm-databases#assignment)


### background

We will create a database to house stream chemistry data (temperature, specific
conductance, dissolved oxygen) collected using data-logging sensors called
sondes. These data were collected from two different locations on three
different days using four different instruments. We will use one table
(`sonde_data`) to store the sensor data, and another (`sonde_events`) to store
data and metadata about each sampling event.  Note that K2\_20 is an
empirically measured estimate of gas exchange across the air-water interface
that is needed for some analyses commonly addressed with this type of data.

We will use SQLite as our database platform, and [DB Browser for
SQLite](http://sqlitebrowser.org/) as a GUI interface to our
database.


### create the database

Creata a new database titled _stream-metabolism_. Upon creating the database,
DB Browser will start with a _Edit table definition_ dialog box. Cancel this
for now.

Before we begin working with our database, we need to indicate that our database
should recognize foreign keys (note that recognizing foreign keys may be the
default option in newer versions of DB Browser). From the _Edit Pragmas_ tab,
check the _Foreign Keys_ option box, then hit _Save_. We will use the defaults
for all other settings.


### database schema

Our database will consist of two tables: (1) `sonde_events` storing data about
sampling details (site, date, instruments, environmental variables), and (2)
`sonde_data` storing water-quality data during a sampling campaign. Our schema
looks like this:

![toy-sonde-schema](/docs/assets/figures/toy-sonde-schema.png)


### create the tables

Use the DB Browser import tool to create the `sonde_events` table by importing
the data: File > Import > Table from CSV file - navigate to the
`sonde_events.csv` file and hit _Open_. You should see the file contents in the
_Import CSV file_ dialog box; name the table 'sonde\_events'. Once created, use
the _Modify Table_ tool to change (1) the id field to Type INTEGER, and check
the NOT NULL, PK (primary key), AI (autoincrement), and U (unique) option boxes,
and (2) the K2\_20 field to type REAL.

Creating the `sonde_data` table is more complicated because we need to (1)
create a compound primary key (see below), and (2) add a FOREIGN KEY that
references the `sonde_events` table. The import tool does not support creating
FOREIGN KEYs so we will need to do this with an SQL statement:

Whereas our sonde\_events data had a single, autoincrementing integer as a
primary key (a common configuration), the sonde data do not lend themselves to
that structure. We will use a composite primary key for the sonde data. A
composite key is a candidate key that consists of two or more attributes (table
columns) that together uniquely identify an entity occurrence (table row).

```sql
CREATE TABLE "sonde_data" (
  "sonde_event_id" INTEGER NOT NULL,
  "Date" TEXT NOT NULL,
  "Time" TEXT NOT NULL,
  "Temp" REAL,
  "SpCond" REAL,
  "DO" REAL,
  PRIMARY KEY ("sonde_event_id", "Date", "Time")
  FOREIGN KEY ("sonde_event_id") REFERENCES sonde_events("id")
);
```

Copy the statement above into the dialog box on the _Execute SQL_ tab, and hit
the play button to execute the command.


### load sonde data into the database

The sonde data are in three different files:

- `sonde_data_1_3.csv`: data corresponding to sonde events 1 through 3
- `sonde_data_4.csv`: ...to sonde event 4
- `sonde_data_5.csv` ...to sonde event 5

Import `sonde_data_1_3.csv` using the import tool being sure to name the import
`sonde_data` so that DB Browser imports the data into the existing table rather
than making a new one based on the file name. Repeat this step for
`sonde_data_4.csv`.

Our FOREIGN KEY references the id field in our `sonde_events` table. Try
importing `sonde_data_5.csv`, which is a fifth sampling event but one that is
not included in our `sonde_events` table, using the aforementioned steps.

Add a fifth event to the `sonde_events` table then try again. We can use DB
Browser's _New Record_ tool and _Browse Data_ interface to add a new record and
to populate it with data, but far preferable and amenable to a reproducible
workflow is to address this with a script like the one below.

```sql
INSERT INTO sonde_events (
  site_id,
  date,
  instrument_id,
  K2_20
)
VALUES (
  "new site",
  "2020-03-20",
  "yellow",
  60.25
);
```

### getting data out of the database

We use **SELECT** statements to extract data from the tables. An asterisk is a
wild card that instructs the query to pull data from all columns.

```sql 
SELECT * 
FROM sonde_data;
```

Or we can extract specific columns:

```sql 
SELECT 
  Date 
FROM sonde_data;
```
Usually we want to use certain search criteria:

```sql 
SELECT
  Date,
  DO
FROM sonde_data 
WHERE strftime('%m', Date) = '08' AND strftime('%d', Date) = '15';
```
We can group results based on data features that can be binned, this is
particularly useful for aggregate functions. The query below extracts the
minimum and maximum dissolved oxygen (DO) values for each sonde\_event.

```sql 
SELECT
  sonde_event_id,
  MIN(DO) AS min_DO,
  MAX(DO) AS max_DO
FROM sonde_data 
GROUP BY sonde_event_id;
```

The ability to link information in our tables is a core features of databases.
We do this with **JOINs**.

For example, incorporate site\_id from the `sonde_events` table into a query of
temperature and dissolved oxygen from `sonde_data`:

```sql 
SELECT
  sonde_events.site_id,
  sonde_data.Temp,
  sonde_data.DO
FROM sonde_data 
JOIN sonde_events ON (sonde_events.id = sonde_data.sonde_event_id)
WHERE sonde_events.id = 1;
```
For example, find the minimum and maximum dissolved oxygen values for each
sonde\_event as per above but this time include the site and the K2\_20 from the
`sonde_events` table:

```sql 
SELECT
  sonde_events.id,
  sonde_events.site_id,
  MIN(sonde_data.DO) AS min_DO,
  MAX(sonde_data.DO) AS max_DO,
  sonde_events.K2_20
FROM sonde_data 
JOIN sonde_events ON (sonde_events.id = sonde_data.sonde_event_id)
GROUP BY sonde_events.id;
```

## assignment

Barometric pressure has a considerable influence on dissolved oxygen
concentrations in aquatic systems. We will need these data for our analyses.

-   create a table in your database to house barometric pressure data
    corresponding to each sonde event. Note that the date and time are
    in a single column in this file, and that BP is barometric pressure.
-   populate the table with data in the atm\_pressure.csv file. The
    table should include these features:
    -   an auto-incrementing primary key
    -   an appropriate data type for each field
    -   a foreign key in which the sonde\_event\_id field of your new
        tables references the id field of the sonde\_events table
-   produce a summary table detailing the sampling site and *average*
    barometric pressue for each sonde event. See
    [here](https://www.sqlite.org/lang_aggfunc.html) for details about
    aggregating functions in SQLite.

Your submission should include the following components:

1.  in a single markdown (md) file:
    -   create table statement generating a table to house the
        barometric pressure data
    -   select statement that yields the sampling site and *average*
        barometric pressue for each sonde event

2.  the output of your select statement as a csv file

**Bonus submission:**

If you want to take it a bit further, generate a query that will produce the
minium and maximum dissolved oxygen values, and the average barometric pressure
for each sonde event. Hint: this requires a join on a select statement. The
structure looks something like this:

    SELECT
      tbl1.field1,
      tbl1.field2,
      subquery.field1
    FROM tbl1
    JOIN (
      SELECT 
        tbl2.field1,
        tbl2.field2
      FROM tbl2
      GROUP BY tbl2.groupingvar
    ) AS subquery ON (subquery.some_id = tbl1.some_id)
    GROUP BY tbl1.groupingvar;

**logistics**: this assignment is not required for the spring 2020 semester.
