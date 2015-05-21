

# Scope of this document #

This documentation was created to accompany the [Public Data Explorer (PDE)](http://www.google.com/publicdata) visualizations hosted at [measurementlab.net/visualization](http://measurementlab.net/visualization). Our aim is to expose the analytic methodology employed when computing these charts from the raw data. In addition, we hope to provide others interested in similar analysis and visualization with the tools they need to replicate and build on what we've done. If anything in this document is unclear, your feedback and questions at [measurementlab.net/contact](http://measurementlab.net/contact) would be greatly appreciated.

# Introduction #

All the charts published at [http://measurementlab.net/visualizations](http://measurementlab.net/visualizations) are computed from the [web100](http://web100.org/) logs collected by [NDT](http://measurementlab.net/measurement-lab-tools#ndt). The web page [https://developers.google.com/bigquery/docs/dataset-mlab](https://developers.google.com/bigquery/docs/dataset-mlab) explains what web100 logs are and how to access them.

The metrics visualized in the charts are computed as follows:
  1. Compute the metric values for every test, by running SQL queries against the [M-Lab tables in BigQuery](https://developers.google.com/bigquery/docs/dataset-mlab).
    * At this step, test results from incomplete tests are discarded.
  1. Geolocate the results of the query.
  1. Aggregate the results of the query by month, geography (city, region, country and world-wide) and ISP.
    * At each level of aggregation, locations with fewer than 200 distinct client IP addresses are not included in the charts.

This page describes these steps in detail.

# Short description of NDT #

This section is only meant to provide enough information about NDT to understand the rest of the document. We have created an illustration to clarify how NDT works.

![http://wiki.m-lab.googlecode.com/git/ndt_explanation.png](http://wiki.m-lab.googlecode.com/git/ndt_explanation.png)

For further information about NDT, see [here](http://measurementlab.net/measurement-lab-tools#ndt).

  * NDT is a client-server application that measures - among other things - the throughput of a single TCP connection, by transferring as much data as possible from the client to the server (client-to-server test) and from the server to client (server-to-client test) for at least 10 seconds, in each direction.
    * Results of NDT tests are indicated in BigQuery with `project = 0`
    * Results of client-to-server tests are indicated in BigQuery with `connection_spec.data_direction = 0`
    * Results of server-to-client tests are indicated in BigQuery with  `connection_spec.data_direction = 1`
  * To estimate the performance of a user connection, NDT attempts to _stress_ the connection, by creating congestion between the user’s machine and an M-Lab server. An NDT test can end in 3 possible states:
    * The test ends during slow start and never reaches congestion. This can happen if the test was interrupted by the user or due to some errors. In this case, the test results underestimate the connection performance, because the test was not able to stress the connection. This condition is expressed in BigQuery with: `web100_log_entry.snap.CongSignals = 0`
    * The test ends after slow start, but during the first congestion episode. This can happen if the test was interrupted by the user or due to some errors. In this case, the peak values in the test results are valid data points, while average values are not. This condition is expressed in BigQuery with:`web100_log_entry.snap.CongSignals = 1`
    * The test ends after the first congestion episode. In this case, the both peak values and averages in the test results are valid data points. This condition is expressed in BigQuery with: `web100_log_entry.snap.CongSignals > 1`
  * During every test, the values of the web100 variables are incremented when TCP-related events occur. NDT polls the values of all the web100 variables every 5ms and writes them into a log file. Every test has a separate log file. The last entry of this log accounts for the whole test and contains the snapshot of web100 variable values that is most commonly used for analysis of NDT tests.
    * Test results extracted from the last line of every web100 log are indicated in BigQuery with `web100_log_entry.is_last_entry = True`
  * The web100 logs used for the analysis are collected only on the server side, for both server-to-client and client-to-server tests. As a consequence, the web100 variables used to study client-to-server tests are not always the same variables used to study server-to-client tests. For example,
    * The duration of a server-to-client test is `web100_log_entry.snap.SndLimTimeRwin + web100_log_entry.snap.SndLimTimeCwnd + web100_log_entry.snap.SndLimTimeSnd`
      * This accounts for the time interval between when the TCP connection was created and the last TCP event recorded.
    * The duration of a client-to-server test is `web100_log_entry.snap.Duration`
      * This accounts for the time interval between when the TCP connection was created and the last time NDT polled the web100 variable values. This may include some idle time at the end of the connection. However, it's not possible to get a more accurate estimate of the duration of client-to-server tests, because `web100_log_entry.snap.SndLimTimeRwin, web100_log_entry.snap.SndLimTimeCwnd, web100_log_entry.snap.SndLimTimeSnd` don't record the duration of client-to-server tests.
    * The data transferred during of a client-to-server test is `web100_log_entry.snap.HCThruOctetsReceived` We use HCThruOctetsReceived rather than ThruOctetsReceived because HCThruOctetsReceived is encoded with 64 bits, as opposed to ThruOctetsReceived which is only encoded with 32 bits and is prone to overflow.
    * The data transferred during of a server-to-client test is `web100_log_entry.snap.HCThruOctetsAcked` We use HCThruOctetsAcked rather than ThruOctetsAcked because HCThruOctetsAcked is  encoded with 64 bits, as opposed to ThruOctetsAcked which is only encoded with 32 bits and is prone to overflow.
  * Most of the web100 variables are incremented when data are sent and not when data are received. As a consequence, it's possible to compute some metrics only for the server-to-client tests and not for the client-to-server tests.

## Incomplete tests ##

In the following, a test is defined **incomplete** if it was interrupted by the user or due to some errors
  * Before the measurement ended, or
  * Before the test completed logging the measurement data.

Complete tests are identified by the following criteria:
  1. The test lasted longer than **9 seconds**.
    * Allowed some flexibility, to account for tests that lasted _almost_ 10 seconds.
    * This condition is expressed in BigQuery
      * For client-to-server tests: `web100_log_entry.snap.Duration >= 9000000`
      * For server-to-client tests: `web100_log_entry.snap.SndLimTimeRwin + web100_log_entry.snap.SndLimTimeCwnd + web100_log_entry.snap.SndLimTimeSnd >= 9000000`
    * We also exclude results from tests that lasted much longer than expected (e.g., 10 min), because this is likely a symptom of problems during the test run.
  1. The test exchanged at least **8192 bytes**.
    * This excludes results from users with dial-up connections and instances where MTU negotiation fails.
    * This condition is expressed in BigQuery
      * For client-to-server tests: `web100_log_entry.snap.HCThruOctetsReceived >= 8192`
      * For server-to-client tests: `web100_log_entry.snap.HCThruOctetsAcked >= 8192`
  1. The test concluded the 3-way-handshake and the connection was **established** (and possibly **closed**).
    * See more information about TCP states and how they are recorded in the web100 logs, see http://www.web100.org/download/kernel/tcp-kis.txt
    * This condition is expressed in BigQuery with: `web100_log_entry.snap.State == 1 || (web100_log_entry.snap.State >= 5  && web100_log_entry.snap.State <= 11)`

# Compute test-level metrics #

For each metric, this section includes the BigQuery queries used to compute the metrics.
All the queries include conditions to filter out incomplete test results.

## Download throughput ##

The download throughput is computed for every server-to-client test as the ratio of the data transmitted during the test and the duration of the test. We multiply by 8 to convert from octets (bytes) to bits:

```
8 *
(web100_log_entry.snap.HCThruOctetsAcked /
 (web100_log_entry.snap.SndLimTimeRwin +
 web100_log_entry.snap.SndLimTimeCwnd +
 web100_log_entry.snap.SndLimTimeSnd))
```

Results of tests that ended during **slow start** are excluded.

The complete BigQuery query is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 8 * (web100_log_entry.snap.HCThruOctetsAcked /
      (web100_log_entry.snap.SndLimTimeRwin +
       web100_log_entry.snap.SndLimTimeCwnd +
       web100_log_entry.snap.SndLimTimeSnd)) AS download_Mbps
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.CongSignals)
 AND web100_log_entry.snap.CongSignals > 0
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

## Upload throughput ##

The upload throughput is computed for every client-to-server test as the ratio of the data transmitted during the test and the duration of the test. We multiply by 8 to convert from octets (bytes) to bits: `8 * (web100_log_entry.snap.HCThruOctetsReceived/web100_log_entry.snap.Duration)`

It is not possible to exclude results of tests that ended during slow start, because the web100 variable `web100_log_entry.snap.CongSignals` is not updated during client-to-server tests.

The complete BigQuery query is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 8 * (web100_log_entry.snap.HCThruOctetsReceived/web100_log_entry.snap.Duration) AS upload_Mbps
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsReceived)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.Duration)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 0
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsReceived >= 8192
 AND web100_log_entry.snap.Duration >= 9000000
 AND web100_log_entry.snap.Duration < 3600000000
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

## Round Trip Time (RTT) ##

Server-to-client RTT is affected by TCP congestion. As a consequence, there are (at least) 2 ways to estimate the RTT using the web100 data. These 2 ways provide different, non-equivalent information about the user connection.
  1. Server-client **(time) distance**
    * Estimated using the **minimum RTT** measured during the test, which most likely happened before the test reached congestion.
    * This value is reported by the web100 variable `web100_log_entry.snap.MinRTT`
    * However, using this variable has the drawback that it might underestimate the connection RTT, because it might be measured in the SYC ACK exchange or some other tiny transaction which, for low speed links, does not represent the typical RTT for the full data segment.
    * Note that using `PreCongSumRTT/PreCongCountRTT` does not provide a more accurate estimate, because both `PreCongSumRTT` and `PreCongCountRTT` are recorded right before the first congestion signal, which, in the worst case, occurs when the receiver queue is already full, which affects the RTT.
  1. Server-client **latency during data transfers** (with congestion)
    * Estimated using the **average RTT**, uniformly averaged over an entire test.
    * This value can be computed as `web100_log_entry.snap.SumRTT/web100_log_entry.snap.CountRTT`
    * In this case, it makes sense to exclude results of tests with fewer than **10 round trip time samples**, because there are not enough samples to accurately estimate the RTT. This condition is expressed in BigQuery with:`web100_log_entry.snap.CountRTT > 10`

Given that the NDT server updates the web100 variables `web100_log_entry.snap.MinRTT` and `web100_log_entry.snap.CountRTT` only when it receives an acknowledgement and given that, during client-to-server tests the NDT server receives an ack only during the 3-way-handshake, RTT values are computed only for server-to-client tests.

The complete BigQuery query is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 web100_log_entry.snap.MinRTT AS min_rtt
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.MinRTT)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.CountRTT)
 AND web100_log_entry.snap.CountRTT > 0
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

## Number of tests ##

The number of tests counts all the server-to-client tests whose results meet the same criteria used to compute the download throughput.
As a consequence, the number of tests is computed as the number of entries in the result of the BigQuery query in the [Download throughput section](#Download_throughput.md).

## Percentage of tests that reached congestion ##

The percentage of tests that reached congestion is computed as the ratio between the number of tests that reached congestion and the number of all the tests, where
  * The number of tests that reached congestion is computed as the number of entries in the result of the BigQuery query in the [Download throughput section](#Download_throughput.md), with the additional condition `web100_log_entry.snap.CongSignals > 1`
    * Given that the NDT server updates the web100 variable `web100_log_entry.snap.CongSignals` only when the server receives a congestion signal, the number of tests that reached congestion is reported for only server-to-client tests.
  * The number of all the tests is computed as described in the [Number of tests section](#Number_of_tests.md).

## Packet retransmission ##

The NDT keeps track of the number of packets retransmitted during a test, in the web100 variable `web100_log_entry.snap.SegsRetrans`.

Packet retransmission is computed as the ratio between the re-transmitted packets and all the transmitted packets.
`web100_log_entry.snap.SegsRetrans/web100_log_entry.snap.DataSegsOut`

Given that the NDT server updates the web100 variables `web100_log_entry.snap.SegsRetrans` and `web100_log_entry.snap.DataSegsOut` only when sending data, packet retransmission is only estimated for server-to-client tests.

It is possible to also measure the byte retransmission, as `web100_log_entry.snap.OctetsRetrans/web100_log_entry.snap.DataOctetsOut`

The complete BigQuery query is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 (web100_log_entry.snap.SegsRetrans / web100_log_entry.snap.DataSegsOut) AS packet_retransmission_rate
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SegsRetrans)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.DataSegsOut)
 AND web100_log_entry.snap.DataSegsOut > 0
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

## Network-limited ratio and client-limited time ratio ##

An NDT test can be in 3 different states. Each state represents different conditions that limit the data sent by the server to the client.
  * **network-limited**, when the network is congested.
    * The web00 variable `web100_log_entry.snap.SndLimTimeCwnd` reports the time spent in this state.
  * **receiver-limited**, when the receiver (client) limits the data that can be received.
    * The web00 variable `web100_log_entry.snap.SndLimTimeRwin` reports the time spent in this state.
  * **server-limited**, when the server limits the data that can be sent.
    * The web00 variable `web100_log_entry.snap.SndLimTimeSnd` reports the time spent in this state.
    * This state can happen because
      * The uplink is congested. Note however that M-Lab servers are specially (over)provisioned to avoid this case.
      * The server cannot send as much data as the network and the receiver would allow, in specific phases of the tests. This usually happens during slow start, especially with fast networks and high values of initial window.
      * The server cannot send as much data as the network and the receiver would allow, during the whole test. This can happen for specific TCP configurations (e.g., if GSO is on). However, this kind of configuration is disabled by default on M-Lab servers.

The complete BigQuery query to compute network-limited time is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip,
 web100_log_entry.connection_spec.local_ip,
 web100_log_entry.snap.SndLimTimeCwnd /
   (web100_log_entry.snap.SndLimTimeRwin +
    web100_log_entry.snap.SndLimTimeCwnd +
    web100_log_entry.snap.SndLimTimeSnd) AS network_limited_time
FROM [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
AND (web100_log_entry.snap.State == 1
OR (web100_log_entry.snap.State >= 5
AND web100_log_entry.snap.State <= 11))
```

The complete BigQuery query to compute receiver-limited time is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 web100_log_entry.snap.SndLimTimeRwini /
   (web100_log_entry.snap.SndLimTimeRwin +
    web100_log_entry.snap.SndLimTimeCwnd +
    web100_log_entry.snap.SndLimTimeSnd) AS receiver_limited_time
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

## Receiver window scale ##

The receiver window scale is the value negotiated at the beginning of a TCP connection to scale the receiver window size. The receive window size is the maximum amount of received data that can be buffered at one time on the receiving side of a TCP connection.
The value of receiver window scale depends on the type and the version of the client's operating system. As a consequence, the distribution of receiver window scale values shows the distribution of operating systems among NDT users.

The receiver window scale of a test is the value of the web100 variable `web100_log_entry.snap.WinScaleRcvd`.
As described in the [web100 variable definition](http://www.web100.org/download/kernel/tcp-kis.txt), the valid values of `web100_log_entry.snap.WinScaleRcvd` are (-1 .. 14), where -1 means that the receiver did not request any value.

The complete BigQuery query is:
```sql
SELECT
 web100_log_entry.connection_spec.remote_ip AS remote_ip,
 web100_log_entry.connection_spec.local_ip AS local_ip,
 web100_log_entry.snap.WinScaleRcvd
FROM
 [plx.google:m_lab.YYYY_MM.all]
WHERE
 IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.remote_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.connection_spec.local_ip)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.HCThruOctetsAcked)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeRwin)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeCwnd)
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.SndLimTimeSnd)
 AND project = 0
 AND IS_EXPLICITLY_DEFINED(connection_spec.data_direction)
 AND connection_spec.data_direction = 1
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.is_last_entry)
 AND web100_log_entry.is_last_entry = True
 AND web100_log_entry.snap.HCThruOctetsAcked >= 8192
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +
      web100_log_entry.snap.SndLimTimeSnd) >= 9000000
 AND (web100_log_entry.snap.SndLimTimeRwin +
      web100_log_entry.snap.SndLimTimeCwnd +  
      web100_log_entry.snap.SndLimTimeSnd) < 3600000000
 AND IS_EXPLICITLY_DEFINED(web100_log_entry.snap.WinScaleRcvd)
 AND web100_log_entry.snap.WinScaleRcvd >= -1
 AND web100_log_entry.snap.WinScaleRcvd <= 14
 AND (web100_log_entry.snap.State == 1
      OR (web100_log_entry.snap.State >= 5
          AND web100_log_entry.snap.State <= 11))
```

# Aggregate query results #

Query results computed as described in the previous section are aggregated using various functions.
This section describes the aggregation functions and how they are applied to the query results to compute the data visualized in the Public Data Explorer charts.

## Dimensions ##

All the metrics are aggregated along the following dimensions:
  * Geography (country, region, city)
  * M-Lab site (see the [list of all the M-Lab sites](http://measurementlab.net/mlab_sites))
  * ISP
    * Every test is assigned to the AS (Autonomous System) that originates the client IP address.
    * All the ASes are aggregated into ISPs, according to the AS names published at [bgp.potaroo.net/cidr/autnums.html bgp.potaroo.net/cidr/autnums.html]. This [file](https://storage.cloud.google.com/m-lab/isp_to_asns_map.txt) lists all the ISPs visualized in the charts and, for each ISP, it lists all the ASes that are assigned to the ISP.

## Aggregation functions ##

### Median ###

Values of download throughput, upload throughput, and RTT are aggregated using the median function.

Note that, at every level of aggregation, the median is computed across the non aggregated values. For example, the median download throughput of country X is computed as the median of all the download throughput values geolocated within the country X (and not as the median of the download throughput medians of all the regions within X).

### Average ###

Values of receiver window scale are aggregated using the average function, weighted by the number of tests.

### Probability ###

Values of packet retrasmission, network-limited time ratio and client-limited time ratio at every level of aggregation are computed as the ration between two sums, computed across all the tests in the same aggregation bin.
In particular:

* Packet retransmission = `SUM(web100_log_entry.snap.SegsRetrans) /
SUM(web100_log_entry.snap.DataSegsOut)`
* Network-limited time ratio = `SUM(web100_log_entry.snap.SndLimTimeCwnd) /
SUM(web100_log_entry.snap.SndLimTimeRwin +
web100_log_entry.snap.SndLimTimeCwnd +
web100_log_entry.snap.SndLimTimeSnd)`
* Client-limited time ratio = `SUM(web100_log_entry.snap.SndLimTimeRwin) /
SUM(web100_log_entry.snap.SndLimTimeRwin +
web100_log_entry.snap.SndLimTimeCwnd +
web100_log_entry.snap.SndLimTimeSnd)`
