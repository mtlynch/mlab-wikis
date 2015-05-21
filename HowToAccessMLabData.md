# How to access the M-Lab data #

All data collected by the M-Lab tools are available to the public without restriction under a [No Rights Reserved Creative Commons Zero Waiver](http://creativecommons.org/about/cc0).
This page describes
  * How to download the data in raw format (i.e., as collected by the M-Lab servers).
  * How to query the M-Lab data via an SQL-like interface, without having to download the data locally.
Both the raw data and the data accessible the SQL interface are updated once a day.

The M-Lab team is also available to release the data via other means. If you have alternative suggestions, please [contact us](http://measurementlab.net/contact).

Every tool logs data in its own format. You can find the information about each dataset and code to parse the raw logs in the [tests page](http://measurementlab.net/tests).

# Download the raw data #

All the M-Lab raw data are organized into tarballs, which are grouped by
  * the **tool** that generated the data,
  * the **date** when the data was collected,
  * the **server** that collected the data.
This means that each tarball contains all the data collected during a single day, by a single tool running on a single M-Lab server.
If the data collected during a day, by one tool on one server are more than 1GB (uncompressed), those data are split into multiple tarballs of up to 1GB size.
For example, the tarball `20090218T000000Z-mlab1-lga01-ndt-0000.tgz` contains the first 1GB of data collected by all the NDT tests that were served by the M-Lab server mlab1-lga01 on Feb 18 2009.

The M-Lab tarballs are stored on [Cloud Storage](https://cloud.google.com/products/cloud-storage). Cloud Storage is Google's cloud storage service (similar to Amazon S3) and provides different ways to access the M-Lab tarballs. In particular, it provides
  * A command line tool (gsutil) to
    * List the content of a folder. `E.g., gsutil ls -l gs://m-lab/ndt/`
    * Download one or more tarballs. `E.g., gsutil cp gs://m-lab/ndt/2009/02/18/20090218T000000Z-mlab1-lga01-ndt-0000.tgz.`
  * A web-based interface to
    * Browse the repository at [https://storage.cloud.google.com/m-lab](https://storage.cloud.google.com/m-lab).
    * Read the complete list of tarballs at [https://storage.cloud.google.com/m-lab/list/all\_mlab\_tarfiles.txt.gz](https://storage.cloud.google.com/m-lab/list/all_mlab_tarfiles.txt.gz).
    * Download each tarball using its public URL `https://storage.cloud.google.com/m-lab/<tool name>/<year>/<month/<day>/<tarball name>`

For example, the file `https://storage.cloud.google.com/m-lab/list/all_mlab_tarfiles.txt.gz` contains the row `gs://m-lab/ndt/2009/02/23/20090223T000000Z-mlab2-lga01-ndt-0000.tgz` that corresponds to a tarball whose url is `https://storage.cloud.google.com/m-lab/ndt/2009/02/23/20090223T000000Z-mlab2-lga01-ndt-0000.tgz`.
Note that it is not possible to use `wget` or `curl` to download the data, as Cloud Storage requires authentication.

# Execute SQL queries against M-Lab data #

M-Lab dataset can be queried using [BigQuery](https://developers.google.com/bigquery/), which allows to efficiently run SQL queries against huge datasets.
Currently, BigQuery contains only a portion of the M-Lab dataset.

More details about the M-Lab data in BigQuery and how to query them can be found [here](https://developers.google.com/bigquery/docs/dataset-mlab).