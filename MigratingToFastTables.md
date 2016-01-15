# Migrating to M-Lab Fast Tables

In January 2016, M-Lab launched new M-Lab BigQuery tables ("fast tables") to public beta testing. This guide walks users through the process of converting their existing BigQuery SQL queries to take advantage of these new tables.

## 1. Change the `FROM` clause to refer to the fast tables

Users refer to M-Lab's existing tables with a clause similar to the following:

```
FROM
  plx.google:m_lab.2016_01.all
```

To use the fast tables, replace the `YYYY_MM` date in the table name with the M-Lab project of interest. For a query over NDT data, this becomes:

```
FROM
  plx.google:m_lab.ndt.all
```

The full list of available fast tables is:

* `plx.google:m_lab.ndt.all`
* `plx.google:m_lab.npad.all`
* `plx.google:m_lab.sidestream.all`
* `plx.google:m_lab.paris_traceroute.all`

## 2. Add a `WHERE` clause to restrict based on test time

If your queries were using the table names as a means of limiting a query to a particular time range, it will be necessary to add `WHERE` clauses to continue enforcing this time range.

For NDT, NPAD, and SideStream, add a query of the following form (where the
numeric values are UNIX timestamps in seconds):

```
WHERE
  (web100_log_entry.log_time >= 1420070400)      -- 2015-01-01T00:00:00Z
   AND (web100_log_entry.log_time < 1427846400)) -- 2015-04-01T00:00:00Z
```

BigQuery also provides a number of date/time formatting functions to convert
human-readable dates into timestamps, for example:

```
WHERE
  (web100_log_entry.log_time >= PARSE_UTC_USEC("2015-01-01 00:00:00") / POW(10, 6))
   AND (web100_log_entry.log_time < PARSE_UTC_USEC("2015-04-01 00:00:00") / POW(10, 6))
```

For Paris Traceroute, add a query of the following form (where the numeric
values are UNIX timestamps in seconds):

```
WHERE
  (log_time >= 1420070400)                       -- 2015-01-01T00:00:00Z
   AND (log_time < 1427846400))                  -- 2015-04-01T00:00:00Z
```

BigQuery also provides a number of date/time formatting functions to convert
human-readable dates into timestamps, for example:

```
WHERE
  (log_time >= PARSE_UTC_USEC("2015-01-01 00:00:00") / POW(10, 6))
   AND (log_time < PARSE_UTC_USEC("2015-04-01 00:00:00") / POW(10, 6))
```

## 3. (optional) Remove `WHERE` clauses for `project`

M-Lab's existing tables combine data for several different M-Lab projects (NDT, NPAD, SideStream, and Paris Traceroute) into the same table. As such, queries for a particular project's data required the query author to add a `WHERE project=XX` clause to restrict the query to a particular project. The new, fast tables are grouped on a per-project basis, so project filters are not necessary.

Within each of the new, fast tables, all rows will contain the same value for `project` and `web100_log_entry.is_last_entry`. Therefore these `WHERE` clauses can be removed.

```
FROM
  plx.google:m_lab.ndt.all
WHERE
  project = 0 -- << Extraneous, all rows in m_lab.ndt have project = 0. Clause
              --    should be deleted.
```

It is not strictly necessary to remove `project` clauses when using the new tables, but doing so will improve query performance.

## 4. (optional) Remove `WHERE` clauses for `is_last_entry`

M-Lab's existing tables include every web100 snapshot collected during the run of each test. The new, fast tables include only the test's final web100 snapshot. As such, clauses to restrict the query to the final snapshot of the test are no longer necessary and can be removed.

```
FROM
  plx.google:m_lab.ndt.all
WHERE
  web100_log_entry.is_last_entry = TRUE -- << Extraneous, all rows in m_lab.ndt
                                        --    have is_last_entry = TRUE. Clause
                                        --    should be deleted.
```

It is not strictly necessary to remove `is_last_entry` clauses when using the new tables, but doing so will improve query performance.

*Note: this does not apply to Paris Traceroute data, as Paris Traceroute is not web100-based, so Paris Traceroute queries never included this clause.*

## Complete Example

To tie it all together, we will look at a complete example where we convert an existing query to take advantage of the new, fast tables.

The query below calculates the total number of NDT tests performed against M-Lab servers for each day in the last quarter of 2015.

### Original query

```
SELECT
  STRFTIME_UTC_USEC(web100_log_entry.log_time * 1000000,
                    '%Y-%m-%d') AS day,
  COUNT(*) AS num_tests
FROM
  [plx.google:m_lab.2015_10.all],
  [plx.google:m_lab.2015_11.all],
  [plx.google:m_lab.2015_12.all]
WHERE
  project = 0
  AND web100_log_entry.is_last_entry = TRUE
  AND web100_log_entry.log_time IS NOT NULL
GROUP BY
  day
ORDER BY
  day ASC
```

### Converted to use fast tables

```
SELECT
  STRFTIME_UTC_USEC(web100_log_entry.log_time * 1000000,
                    '%Y-%m-%d') AS day,
  COUNT(*) AS num_tests
FROM
  [plx.google:m_lab.ndt.all]
WHERE
  ((web100_log_entry.log_time >= 1443657600)     -- 2015-10-01T00:00:00Z
   AND (web100_log_entry.log_time < 1451606400)) -- 2016-01-01T00:00:00Z
GROUP BY
  day
ORDER BY
  day ASC
```

The converted query performs significantly faster than the original:

| Query Type                   | Observed Execution Time |
|------------------------------|-------------------------|
| Original query               | 129.1 seconds           |
| Converted to use fast tables | 5.8 seconds             |

## Questions / Feedback

If you have questions or feedback about using the new, fast tables, please send an email to [support@measurementlab.net](support@measurementlab.net).
