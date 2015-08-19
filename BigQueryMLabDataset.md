# M-Lab Dataset - BigQuery — Google Cloud Platform

## Contents

## About

[Measurement Lab][1] (M-Lab) is an open, distributed server platform that researchers can use to deploy tools that measure broadband connection performance. Anybody can use these tools to measure their own connection performance. The M-Lab servers collect logs of these user tests. These logs are available to the public without restriction under a [No Rights Reserved Creative Commons Zero Waiver][2].

BigQuery contains M-Lab logs generated since **January 2009** by three M-Lab tools, [Network Diagnostic Tool][3] (NDT), [Network Path and Application Diagnosis][4] (NPAD), [SideStream][5], and [Paris-Traceroute][6]. BigQuery tables are updated every day with data from M-Lab logs collected the day before. As a consequence, there is typically less than a 24-hour delay between data collection and data publication.

**Name:** The M-Lab BigQuery dataset contains many tables. To see a list of available M-Lab log tables, run `bq ls measurement-lab:m_lab` using the [bq command-line tool][7]. Then, to access a certain table, specify the project:dataset.table identifier as `measurement-lab:m_lab._tableId_` where `tableId` is the name of one of the tables from the list.

**Number of rows:** Many, many. The March 2010 table alone has almost 26 billions rows.

For more details about M-Lab, NDT and NPAD, see the [M-Lab website][8].


## Schema

* Each M-Lab tool consists of a **client** and a **server **. Whenever an M-Lab user starts a test, the client and server interact to measure different aspects of that user's connection.
* A single user request triggers one or more **tests** (e.g., client-to-server test, server-to-client test).
* For each test, a server collects a **Web100 log** and the test can be identified by the log filename.
* A Web100 log is a sequence of **Web100 snapshots**, where a Web100 snapshot consists of the values of all the **Web100 variables** at a given time. The last entry of a Web100 log contains the value of all the Web100 variables at the end of a test.
* BigQuery stores all the M-Lab data in multiple tables, within a **single, public BigQuery dataset**.
* Each table row represents a **single Web100 snapshot** collected during a test.
* For each test, the table contains **one or more rows** (one row for each Web100 snapshot collected during that test).

The following table is an example of the BigQuery M-Lab table. Rows ` a1`, `a2`, `a3` have been collected during the test `a`, while rows `b1` and `b2` have been collected during the test `b`. The next section explains what each field represents.

|               |               |  test_id |  web100_log_entry.log_time |  web100_log_entry.snap.MinRTT |  ... |
| --------------|:-------------:| :-------:|:--------------------------:| :----------------------------:|-----:|
| test `a`      |  snapshot `a1`|  xxxxx   |  1265249248 		             |  8             		             |      |
|               |  snapshot `a2`|  xxxxx   |  1265249248                |  90                           |      |
| 	             |  snapshot `a3`|  xxxxx   |  1265249248                |  137                          |      |
| test `b`      |  snapshot `b1`|  yyyyy   |  1266216058                |  20                           |      |
|               |  snapshot `b2`|  yyyyy   |  1266216058                |  154                          |      |

The following table describes the schema of the BigQuery table that contains M-Lab data.

| Field name             			                  |     Type     |  Description                              |
| :----------------------------------------------------|:------------:|:------------------------------------------|
| `test_id`                                           |  `string`    |  ID of the test. It represents the filename of the log that contains the data generated during the test (e.g., 20090819T02:01:04.507508000Z_189.6.232.77:3859.c2s_snaplog.gz). |
| `project`                                           |  `integer`   |  Tool that ran the test. {NDT = `0`, NPAD = `1`, SideStream = `2`, paris-traceroute = `3`}  |
| `type`                                              |  `integer`   |  Currently not supported. |
| `log_time`                                          |  `integer`   |  Time when the Web100 log was created (in seconds since the Unix epoch, UTC). (This field is **optional**. It's preferable to use `web100_log_entry.log_time`.) |
| `connection_spec.data_direction`                    |  `integer`   |  Direction of the data sent during the test. {CLIENT_TO_SERVER = `0`, SERVER_TO_CLIENT = `1`} |
| `connection_spec.server_ip`                         |  `string`    |  Server's IP address. (This field is **optional**. It's preferable to use <br>
`web100_log_entry.connection_spec.local_ip`.) |
| `connection_spec.server_af`                         |  `integer`   |  Address family of the server's IP address. (This field is **optional**. It's preferable to use `web100_log_entry.connection_spec.local_af`.) |
| `connection_spec.server_hostname`                   |  `string`    |  Server's hostname. (This field is **optional**.) |
| `connection_spec.server_kernel_version`             |  `string`    |  Server's kernel version. (This field is **optional**.) |
| `connection_spec.client_ip `                        |  `string`    |  IP address of the user's client. (This field is **optional**. It's preferable to use `web100_log_entry.connection_spec.remote_ip`.) |
| `connection_spec.client_af`                         |  `integer`   |  Address family of the client's IP address. (This field is **optional**.) |
| `connection_spec.client_hostname`                   |  `string`    |  Client's hostname. (This field is **optional**.) |
| `connection_spec.client_application`                |  `string`    |  Client application that ran the test. (This field is **optional**.) |
| `connection_spec.client_browser`                    |  `string`    |  Client's browser. (This field is **optional**.) |
| `connection_spec.client_os`                         |  `string`    |  Client's operating system. (This field is **optional**.) |
| `connection_spec.client_kernel_version`             |  `string`    |  Client's kernel version. (This field is **optional**.) |
| `connection_spec.client_version`                    |  `string`    |  Client's version. (This field is **optional**.) |
| `connection_spec.client_geolocation.continent_code` |  `string`    |  Geolocation fields extracted from open dataset created by MaxMind<br>
 and available at [www.maxmind.com][9]. (These fields are **optional**.) |
| `connection_spec.client_geolocation.country_code`   |  `string`    |   |
| `connection_spec.client_geolocation.country_code3`  |  `string`    |   |
| `connection_spec.client_geolocation.country_name`   |  `string`    |   |
| `connection_spec.client_geolocation.region`         |  `string`    |   |
| `connection_spec.client_geolocation.metro_code`     |  `integer`   |   |
| `connection_spec.client_geolocation.city`           |  `string`    |   |
| `connection_spec.client_geolocation.area_code`      |  `integer`   |   |
| `connection_spec.client_geolocation.postal_code`    |  `string`    |   |
| `connection_spec.client_geolocation.latitude`       |  `float`     |   |
| `connection_spec.client_geolocation.longitude`      |  `float`     |   |
| `connection_spec.server_geolocation.continent_code` |  `string`    |   |
| `connection_spec.server_geolocation.country_code`   |  `string`    |   |
| `connection_spec.server_geolocation.country_code3`  |  `string`    |   |
| `connection_spec.server_geolocation.country_name`   |  `string`    |   |
| `connection_spec.server_geolocation.region`         |  `string`    |   |
| `connection_spec.server_geolocation.metro_code`     |  `integer`   |   |
| `connection_spec.server_geolocation.city`           |  `string`    |   |
| `connection_spec.server_geolocation.area_code`      |  `integer`   |   |
| `connection_spec.server_geolocation.postal_code`    |  `string`    |   |
| `connection_spec.server_geolocation.latitude`       |  `float`     |   |
| `connection_spec.server_geolocation.longitude`      |  `float`     |   |
| `web100_log_entry.version`                          |  `string`    |  Web100 kernel patch version running on the server (as defined in <br>/proc/web100/header). |
| `web100_log_entry.log_time`                         |  `integer`   |  Time when the Web100 log was created (in seconds since the Unix epoch, UTC). |
| `web100_log_entry.is_last_entry`                    |  `bool`      |  Is this the last entry of this Web100 log file? |
| `web100_log_entry.group_name`                       |  `string`    |  Web100 group name (not supported by the current Web100 implementation).  |
| `web100_log_entry.connection_spec.local_ip`         |  `string`    |  IP address of the M-Lab server, as logged in the Web100 log. |
| `web100_log_entry.connection_spec.local_af`         |  `integer`   |  Address family of the server's IP address, as logged in the Web100 log.  |
| `web100_log_entry.connection_spec.local_port`       |  `integer`   |  Port of the M-Lab server (in host-byte-order), as logged in the Web100 log. |
| `web100_log_entry.connection_spec.remote_ip`        |  `string`    |  IP address of the user's client, as logged in the Web100 log. |
| `web100_log_entry.connection_spec.remote_port`      |  `integer`   |  Port of the user's client (in host-byte-order), as logged in the Web100 log. |
| `web100_log_entry.snap.[web100_var_name]`, where `web100_var_name` is the name of a Web100 variable, as defined in [tcp-kis.txt][10] (field `VariableName`). |  See Web100 types |  [tcp-kis.txt][10] defines 150 Web100 variables. For example, `web100_log_entry.snap.MinRTT` represents the minimum sampled Round Trip Time. |
| `paris_traceroute_hop.protocol`                     |  `integer`   |  Protocol used to generate the paris-traceroute trace. {UDP = `0`, TCP = `1`, ICMP = `2`} |
| `paris_traceroute_hop.src_ip`                       |  `string`    |  The IP address of the start of the hop. |
| `paris_traceroute_hop.src_af`                       |  `integer`   |  The address family used to connect to `src_ip`. {AF_INET = `2`, AF_INET6 = `10`. |
| `paris_traceroute_hop.src_hostname`                 |  `string`    |  The hostname of the start of the hop. This may be the same as `src_ip` if the hostname could not be resolved. |
| `paris_traceroute_hop.dest_ip`                      |  `string`    |  The IP address of the end of the hop. |
| `paris_traceroute_hop.dest_af`                      |  `integer`   |  The address family used to connect to `dest_ip`. {AF_INET = `2`, AF_INET6 = `10`. |
| `paris_traceroute_hop.dest_hostname`                |  `string`    |  The hostname of the end of the hop. This may be the same as `dest_ip` if the hostname could not be resolved. |
| `paris_traceroute_hop.rtt`                          |  `float`     |  The RTT measured from `connection_spec.server_ip` to `paris_traceroute_hop.dest_ip`. |


### Equivalent BigQuery and Web100 Field Types

[tcp-kis.txt][10] defines each Web100 variable with a specific [SNMP type ][11]. This table shows how to map each SNMP type to a BigQuery type.

| BigQuery Type |  Corresponding SNMP Type |
| ------------- | -------------------------| 
| `integer`     |  `Integer32`, `Integer`, `INTEGER`, `Gauge32`, `ZeroBasedCounter32`, `Unsigned32`, `Unsigned16`, `Counter32`, `ZeroBasedCounter64` |
| `string`      |  `Ip_Address`            |
| `bool`        |  `TruthValue`            |


## Sample queries — _Basic statistics about the NDT and NPAD user population_

This section shows you how to compute some basic statistics about the population of users who have run NDT and/or NPAD tests. For purposes of this exercise, each user has a single IP address and each IP address identifies a single user.

In order to limit the query run time, the examples presented in this exercise refer to the data collected by all the M-Lab servers during a very limited time interval (between **midnight January 1st 2010** and **midnight January 3rd 2010**).

Sample queries:

#### Basic counting — How many users?

Let's start with something simple! How many users have ever run **any** test?
* The following query counts the number of unique IP addresses of clients that have ever run at least one test (between midnight Jan 1 2010 and midnight Jan 3 2010).
* The query only takes into account rows where `web100_log_entry.log_time` and `web100_log_entry.connection_spec.remote_ip` are defined. The [ BigQuery Query Reference][12] describes the `IS_EXPLICITLY_DEFINED` function.
* `1262304000` is Jan 1 2010 00:00:00 in Unix time, UTC. `1262476800` is Jan 3 2010 00:00:00 in Unix time, UTC.
``` 
    SELECT COUNT(DISTINCT web100_log_entry.connection_spec.remote_ip) AS num_clients
    FROM [plx.google:m_lab.2010_01.all]
    WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
          IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
          web100_log_entry.log_time > 1262304000 AND
          web100_log_entry.log_time < 1262476800;
``` 
Result:

    num_clients
    -----------
          31960

If you are only interested in the clients that have run **NDT** (`project = 0`) tests, the query is:

    SELECT COUNT(DISTINCT web100_log_entry.connection_spec.remote_ip) AS num_clients
    FROM [plx.google:m_lab.2010_01.all]
    WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
          IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
          web100_log_entry.log_time > 1262304000 AND
          web100_log_entry.log_time < 1262476800 AND
          project = 0;

If you prefer to handle timestamps in a "readable" format, re-write the query as follows:

    SELECT COUNT(DISTINCT web100_log_entry.connection_spec.remote_ip) AS num_clients
    FROM [plx.google:m_lab.2010_01.all]
    WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
          IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
          web100_log_entry.log_time > PARSE_UTC_USEC('2010-01-01 00:00:00') / POW(10, 6) AND
          web100_log_entry.log_time < PARSE_UTC_USEC('2010-01-03 00:00:00') / POW(10, 6);

Be aware that using `PARSE_UTC_USEC` can slow down the query. The [ BigQuery Query Reference][13] describes the `PARSE_UTC_USEC` function.


#### Computing statistics over time — How many users per day?

By slightly modifying the previous query, it is possible to compute how the number of users changed over time.

* The following query splits the time period (between midnight Jan 1 2010 and midnight Jan 3 2010) into 1-day time intervals and counts the number of unique client IP addresses within each time interval. Every client IP address is counted at most once per day.
* `UTC_USEC_TO_DAY` expects a timestamp in microseconds, while `web100_log_entry.log_time` is in seconds. The [ BigQuery Query Reference][13] describes the `UTC_USEC_TO_DAY` function.
``` 
    SELECT INTEGER(UTC_USEC_TO_DAY(web100_log_entry.log_time * 1000000)/1000000) AS day,
          COUNT(DISTINCT web100_log_entry.connection_spec.remote_ip) AS num_clients
    FROM [plx.google:m_lab.2010_01.all]
    WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
          IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
          web100_log_entry.log_time > 1262304000 AND
          web100_log_entry.log_time < 1262476800
    GROUP BY day
    ORDER BY day ASC;
``` 
Result:

    day        num_clients
    ---------- -----------
    1262304000       14489
    1262390400       18266

If you prefer to have the output with timestamps in a "readable" format, use the following query.

    SELECT **STRFTIME_UTC_USEC(day, '%Y-%m-%d') AS day,
           num_clients**
    FROM (
      SELECT UTC_USEC_TO_DAY(web100_log_entry.log_time * 1000000) AS day,
      COUNT(DISTINCT web100_log_entry.connection_spec.remote_ip) AS num_clients
      FROM [plx.google:m_lab.2010_01.all]
      WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
            IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
            web100_log_entry.log_time > 1262304000 AND
            web100_log_entry.log_time < 1262476800
      GROUP BY day
      ORDER BY day ASC
    );

Result:

    day        num_clients
    ---------- -----------
    2010-01-01       14489
    2010-01-02       18266

Be aware that using `STRFTIME_UTC_USEC` can significantly slow down the query. The [BigQuery Query Reference][13] describes the `STRFTIME_UTC_USEC` function.


#### Dealing with IP addresses — How many users from distinct subnets?

BigQuery supports various functions to parse IP addresses in different formats. You can use such functions to aggregate the number of users per subnet and compute how many subnets have ever initiated a test.

* The following query aggregates the client IP addresses into /24s and counts the number of unique /24s that have ever initiated at least one test (between midnight Jan 1 2010 and midnight Jan 3 2010).
* `PARSE_IP(remote_ip) & INTEGER(POW(2, 32) - POW(2, 32 - 24)))` computes a bit-wise AND between `web100_log_entry.connection_spec.remote_ip` and 255.255.255.0. The [ BigQuery Query Reference][14] describes the `PARSE_IP` and `FORMAT_IP` functions.
``` 
    SELECT COUNT(DISTINCT FORMAT_IP(PARSE_IP(web100_log_entry.connection_spec.remote_ip)
                          & INTEGER(POW(2, 32) - POW(2, 32 - 26)))) AS num_subnets
    FROM [plx.google:m_lab.2010_01.all]
    WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
          IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
          web100_log_entry.log_time > 1262304000 AND
          web100_log_entry.log_time < 1262476800;
``` 
Result:

    num_subnets
    -----------
          30256


#### Comparing NDT and NPAD tests — How many users have run both NDT and NPAD tests?

* The following query computes the number of distinct IP addresses that have run tests using 2 distinct tool clients (between midnight Jan 1 2010 and midnight Jan 3 2010).
* The nested query returns a table, where each row represents a client IP address and contains the number of tools ever used by that address.
``` 
    **SELECT COUNT(remote_ip) AS num_ip_addresses
    FROM (
      SELECT web100_log_entry.connection_spec.remote_ip AS remote_ip,
             COUNT(DISTINCT project) AS num_projects**
      FROM [plx.google:m_lab.2010_01.all]
      WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
            IS_EXPLICITLY_DEFINED(project) AND
            IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
            web100_log_entry.log_time > 1262304000 AND
            web100_log_entry.log_time < 1262476800
      GROUP BY remote_ip
    )
    WHERE num_projects = 2;
``` 
Result:

    num_ip_addresses
    ----------------
                  71


#### Computing distributions of tests across users — How many users have run a certain number of tests?

Now let's try something a bit more complex.

Some IP addresses may have initiated tests, while others only have a few tests. To assess the impact of each IP address, you can classify the IP addresses based on the number of tests they have initiated.

* The following query computes the number of tests initiated by each client IP address, groups the IP addresses by the number of tests run, and returns the number of IP addresses in each group.
* The innermost nested query returns a table, where each row represents a test. BigQuery does not support the SQL command `DISTINCT` on multiple fields. So the query uses the `GROUP BY` clause to collapse all the rows with the same `test_id` and `remote_ip`. The [ BigQuery Query Reference][15] describes the `GROUP BY` command.
* The outermost nested query returns a table, where each row represents a client IP address and contains the number of tests initiated by that address.
``` 
    SELECT num_tests,
           COUNT(*) AS num_clients
      FROM (
        SELECT remote_ip,
               COUNT(*) AS num_tests
        FROM (
          SELECT test_id,
                 web100_log_entry.connection_spec.remote_ip AS remote_ip
          FROM [plx.google:m_lab.2010_01.all]
          WHERE IS_EXPLICITLY_DEFINED(web100_log_entry.log_time) AND
                IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip) AND
                web100_log_entry.log_time > 1262304000 AND
                web100_log_entry.log_time < 1262476800 AND
                IS_EXPLICITLY_DEFINED(web100_log_entry.log_time)
          GROUP BY test_id, remote_ip
      )
      GROUP BY remote_ip
    )
    GROUP BY num_tests
    ORDER BY num_tests ASC;
``` 
Result:

    num_tests num_clients
    --------- -----------
            1        6139
            2       15088
            3        2998
            4        2965
            5         985
            6         997
            7         436
            8         430
            9         216
           10         224
           11         158
    [...]



[1]: http://www.measurementlab.net/about
[2]: http://creativecommons.org/about/cc0
[3]: http://www.measurementlab.net/tools/ndt
[4]: http://www.measurementlab.net/tools/npad
[5]: http://www.measurementlab.net/tools/sidestream
[6]: http://www.measurementlab.net/tools/paris-traceroute
[7]: https://cloud.google.com/bigquery/bq-command-line-tool
[8]: http://www.measurementlab.net/
[9]: http://www.maxmind.com/
[10]: https://cloud.google.com/bigquery/docs/tcp-kis.txt
[11]: http://tools.ietf.org/html/rfc4898
[12]: https://cloud.google.com/bigquery/query-reference#comparisonfunctions
[13]: https://cloud.google.com/bigquery/docs/query-reference.html#timestampfunctions
[14]: https://cloud.google.com/bigquery/query-reference#timestampfunctions
[15]: https://cloud.google.com/bigquery/docs/query-reference#groupby
