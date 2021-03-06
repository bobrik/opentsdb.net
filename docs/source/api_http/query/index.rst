/api/query
==========

Probably the most useful endpoint in the API, ``/api/query`` enables extracting data from the storage system in various formats determined by the serializer selected. Queries can be submitted via the 1.0 query string format or body content.

Query API Endpoints
-------------------

.. toctree::
   :maxdepth: 1
   
   last

The ``/query`` endpoint is documented below.

Verbs
-----

* GET
* POST

Requests
--------

Request parameters include:

.. csv-table::
   :header: "Name", "Data Type", "Required", "Description", "Default", "QS", "RW", "Example"
   :widths: 10, 5, 5, 45, 10, 5, 5, 15
   
   "start", "String, Integer", "Required", "The start time for the query. This can be a relative or absolute timestamp. See :doc:`../../user_guide/query/index` for details.", "", "start", "", "1h-ago"
   "end", "String, Integer", "Optional", "An end time for the query. If not supplied, the TSD will assume the local system time on the server. This may be a relative or absolute timestamp. See :doc:`../../user_guide/query/index` for details.", "*current time*", "end", "", "1s-ago"
   "queries", "Array", "Required", "One or more sub queries used to select the time series to return. These may be metric ``m`` or TSUID ``tsuids`` queries", "", "m or tsuids", "", "*See below*"
   "noAnnotations", "Boolean", "Optional", "Whether or not to return annotations with a query. The default is to return annotations for the requested timespan but this flag can disable the return. This affects both local and global notes and overrides ``globalAnnotations``", "false", "no_annotations", "", "false"
   "globalAnnotations", "Boolean", "Optional", "Whether or not the query should retrieve global annotations for the requested timespan", "false", "global_annotations", "", "true"
   "msResolution", "Boolean", "Optional", "Whether or not to output data point timestamps in milliseconds or seconds. If this flag is not provided and there are multiple data points within a second, those data points will be down sampled using the query's aggregation function.", "false", "ms", "", "true"
   "showTSUIDs", "Boolean", "Optional", "Whether or not to output the TSUIDs associated with timeseries in the results. If multiple time series were aggregated into one set, multiple TSUIDs will be returned in a sorted manner", "false", "show_tsuids", "", "true"

Sub Queries
^^^^^^^^^^^

An OpenTSDB query requires at least one sub query, a means of selecting which time series should be included in the result set. There are two types:

* **Metric Query** - The full name of a metric is supplied along with an optional list of tags. This is optimized for aggregating multiple time series into one result.
* **TSUID Query** - A list of one or more TSUIDs that share a common metric. This is optimized for fetching individual time series where aggregation is not required.

A query can include more than one sub query and any mixture of the two types. When submitting a query via content body, if a list of TSUIDs is supplied, the metric and tags for that particular sub query will be ignored.

Each sub query can retrieve individual or groups of timeseries data, performing aggregation or grouping calculations on each set. Fields for each sub query include:

.. csv-table::
  :header: "Name", "Data Type", "Required", "Description", "Default", "Example"
  :widths: 10, 10, 5, 50, 10, 15
  
  "aggregator", "String", "Required", "The name of an aggregation function to use. See :doc:`../aggregators`", "", "sum"
  "metric", "String", "Required", "The name of a metric stored in the system", "", "sys.cpu.0"
  "rate", "Boolean", "Optional", "Whether or not the data should be converted into deltas before returning. This is useful if the metric is a continously incrementing counter and you want to view the rate of change between data points.", "false", "true"
  "rateOptions", "Map", "Optional", "Monotonically increasing counter handling options", "*See below*", "*See below*"
  "downsample", "String", "Optional", "An optional downsampling function to reduce the amount of data returned.", "", "5m-avg"
  "tags", "Map", "Optional", "To drill down to specific timeseries or group results by tag, supply one or more map values in the same format as the query string. Tags are converted to filters in 2.2. See the notes below about conversions. Note that if no tags are specified, all metrics in the system will be aggregated into the results. *Deprecated in 2.2*", "", "*See Below*"
  "filters *(2.2)*", "List", "Optional", "Filters the time series emitted in the results. Note that if no filters are specified, all time series for the given metric will be aggregated into the results.", "", "*See Below*"

*Rate Options*

When passing rate options in a query string, the options must be enclosed in curly braces. For example:  ``m=sum:rate{counter,,1000}:if.octets.in``. If you wish to use the default ``counterMax`` but do want to supply a ``resetValue``, you must add two commas as in the previous example. Additional fields in the ``rateOptions`` object include the following:

.. csv-table::
  :header: "Name", "Data Type", "Required", "Description", "Default", "Example"
  :widths: 10, 10, 5, 50, 10, 15
  
  "counter", "Boolean", "Optional", "Whether or not the underlying data is a monotonically increasing counter that may roll over", "false", "true"
  "counterMax", "Integer", "Optional", "A positive integer representing the maximum value for the counter.", "Java Long.MaxValue", "65535"
  "resetValue", "Integer", "Optional", "An optional value that, when exceeded, will cause the aggregator to return a ``0`` instead of the calculated rate. Useful when data sources are frequently reset to avoid spurious spikes.", "0", "65000"

*Filters*

New for 2.2, OpenTSDB includes expanded and plugable filters across tag key and value combinations. For a list of filters loaded in the TSD, see :doc:`../config/filters`. For descriptions of the built-in filters see :doc:`../../user_guide/query/filters`. Filters can be used in both query string and POST formatted queries. Multiple filters on the same tag key are allowed and when processed, they are *ANDed* together e.g. if we have two filters ``host=literal_or(web01)`` and ``host=literal_or(web02)`` the query will always return empty. If two or more filters are included for the same tag key and one has group by enabled but another does not, then group by will effectively be true for all filters on that tag key. Fields for POST queries pertaining to filters include:

.. csv-table::
  :header: "Name", "Data Type", "Required", "Description", "Default", "Example"
  :widths: 10, 10, 5, 50, 10, 15
  
  "type", "String", "Required", "The name of the filter to invoke. See :doc:`../config/filters`", "", "regexp"
  "tagk", "String", "Required", "The tag key to invoke the filter on", "", "host"
  "filter", "String", "Required", "The filter expression to evaluate and depends on the filter being used", "", "web.*.mysite.com"
  "groupBy", "Boolean", "Optional", "Whether or not to group the results by each value matched by the filter. By default all values matching the filter will be aggregated into a single series.", "false", "true"

For URI queries, the type precedes the filter expression in parentheses. The format is ``<tagk>=<type>(<filter_expression>)``. Whether or not results are grouped depends on which curly bracket the filter is in. Two curly braces are now supported per metric query. The first set is the *group by* filter and the second is a *non group by* filter, e.g. ``{host=web01}{colo=regexp(sjc.*)}. If you only want to filter without grouping then the first curly set must be empty, e.g. ``{}{host=web*}``

.. NOTE:: Regular expression, wildcard filters with a pre/post/in-fix or literal ors with many values can cause queries to return slower as each row of data must be resolved to their string values then processed.

*Filter Conversions*

Values in the POST query ``tags`` map and the *group by* curly brace of URI queries are automatically converted to filters to provide backwards compatibility with existing systems. The auto conversions include:

.. csv-table::
  :header: "Example", "Description"
  :widths: 25, 75
  
  "``<tagk>=*``", "Wildcard filter, effectively makes sure the tag key is present in the series"
  "``<tagk>=value``", "Case sensitive literal OR filter"
  "``<tagk>=value1|value2|valueN``", "Case sensitive literal OR filter"
  "``<tagk>=va*``", "Case insensitive wildcard filter. An asterisk (star) with any other strings now becomes a wildcard filter shortcut"

Metric Query String Format
^^^^^^^^^^^^^^^^^^^^^^^^^^

The full specification for a metric query string sub query is as follows:

::

  m=<aggregator>:[rate[{counter[,<counter_max>[,<reset_value>]]]}:][<down_sampler>:]<metric_name>[{<tag_name1>=<grouping filter>[,...<tag_nameN>=<grouping_filter>]}][{<tag_name1>=<non grouping filter>[,...<tag_nameN>=<non_grouping_filter>]}]
  
It can be a little daunting at first but you can break it down into components. If you're ever confused, try using the built-in GUI to plot a graph the way you want it, then look at the URL to see how the query is formatted. Changes to any of the form fields will update the URL (which you can actually copy and paste to share with other users). For examples, please see :doc:`../../user_guide/query/examples`.

TSUID Query String Format
^^^^^^^^^^^^^^^^^^^^^^^^^

TSUID queries are simpler than Metric queries. Simply pass a list of one or more hexadecimal encoded TSUIDs separated by commas:

::

  tsuid=<aggregator>:<tsuid1>[,...<tsuidN>]
  
Example Query String Requests
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  http://localhost:4242/api/query?start=1h-ago&m=sum:rate:proc.stat.cpu{host=foo,type=idle}
  http://localhost:4242/api/query?start=1h-ago&tsuid=sum:000001000002000042,000001000002000043

Example Content Request
^^^^^^^^^^^^^^^^^^^^^^^

Please see the serializer documentation for request information: :doc:`../serializers/index`. The following examples pertain to the default JSON serializer.

.. code-block :: javascript

  {
      "start": 1356998400,
      "end": 1356998460,
      "queries": [
          {
              "aggregator": "sum",
              "metric": "sys.cpu.0",
              "rate": "true",
              "tags": {
                  "host": "*",
                  "dc": "lga"
              }
          }, 
          {
              "aggregator": "sum",
              "tsuids": [
                  "000001000002000042",
                  "000001000002000043"
                ]
              }
          }
      ]
  }

2.2 query with filters

.. code-block :: javascript

  {
      "start": 1356998400,
      "end": 1356998460,
      "queries": [
          {
              "aggregator": "sum",
              "metric": "sys.cpu.0",
              "rate": "true",
              "filters": [
                  {
                     "type":"wildcard",
                     "tagk":"host",
                     "filter":"*",
                     "groupBy":true
                  },
                  {
                     "type":"literal_or",
                     "tagk":"dc",
                     "filter":"lga|lga1|lga2",
                     "groupBy":false
                  },
              ]
          }, 
          {
              "aggregator": "sum",
              "tsuids": [
                  "000001000002000042",
                  "000001000002000043"
                ]
              }
          }
      ]
  }

Response
--------
   
The output generated for a query depends heavily on the chosen serializer :doc:`../serializers/index`. A request may result in multiple sets of data returned, particularly if the request included multiple queries or grouping was requested. Some common fields included with each data set in the response will be:

.. csv-table::
  :header: "Name", "Description"
  :widths: 20, 80
  
  "metric", "Name of the metric retreived for the time series"
  "tags", "A list of tags only returned when the results are for a single time series. If results are aggregated, this value may be null or an empty map"
  "aggregatedTags", "If more than one timeseries were included in the result set, i.e. they were aggregated, this will display a list of tag names that were found in common across all time series."
  "dps", "Retrieved data points after being processed by the aggregators. Each data point consists of a timestamp and a value, the format determined by the serializer."
  "annotations", "If the query retrieved annotations for timeseries over the requested timespan, they will be returned in this group. Annotations for every timeseries will be merged into one set and sorted by ``start_time``. Aggregator functions do not affect annotations, all annotations will be returned for the span."
  "globalAnnotations", "If requested by the user, the query will scan for global annotations during the timespan and the results returned in this group"

Unless there was an error with the query, you will generally receive a ``200`` status with content. However if your query couldn't find any data, it will return an empty result set. In the case of the JSON serializer, the result will be an empty array:

.. code-block :: javascript  

  []

For the JSON serializer, the timestamp will always be a Unix epoch style integer followed by the value as an integer or a floating point. For example, the default output is ``"dps"{"<timestamp>":<value>}``. By default the timestamps will be in seconds. If the ``msResolution`` flag is set, then the timestamps will be in milliseconds.

Example Aggregated Default Response
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: javascript

  [
      {
          "metric": "tsd.hbase.puts",
          "tags": {},
          "aggregatedTags": [
              "host"
          ],
          "annotations": [
              {
                  "tsuid": "00001C0000FB0000FB",
                  "description": "Testing Annotations",
                  "notes": "These would be details about the event, the description is just a summary",
                  "custom": {
                      "owner": "jdoe",
                      "dept": "ops"
                  },
                  "endTime": 0,
                  "startTime": 1365966062
              }
          ],
          "globalAnnotations": [
              {
                  "description": "Notice",
                  "notes": "DAL was down during this period",
                  "custom": null,
                  "endTime": 1365966164,
                  "startTime": 1365966064
              }
          ],
          "tsuids": [
              "0023E3000002000008000006000001"
          ],
          "dps": {
              "1365966001": 25595461080,
              "1365966061": 25595542522,
              "1365966062": 25595543979,
  ...
              "1365973801": 25717417859
          }
      }
  ]

Example Aggregated Array Response
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: javascript

  [
      {
          "metric": "tsd.hbase.puts",
          "tags": {},
          "aggregatedTags": [
              "host"
          ],
          "dps": [
              [
                  1365966001,
                  25595461080
              ],
              [
                  1365966061,
                  25595542522
              ],
  ...
              [
                  1365974221,
                  25722266376
              ]
          ]
      }
  ]
  
Example Multi-Set Response
^^^^^^^^^^^^^^^^^^^^^^^^^^

For the following example, two TSDs were running and the query was: ``http://localhost:4242/api/query?start=1h-ago&m=sum:tsd.hbase.puts{host=*}``. This returns two explicit time series.

.. code-block:: javascript

  [
      {
          "metric": "tsd.hbase.puts",
          "tags": {
              "host": "tsdb-1.mysite.com"
          },
          "aggregatedTags": [],
          "dps": {
              "1365966001": 3758788892,
              "1365966061": 3758804070,
  ...
              "1365974281": 3778141673
          }
      },
      {
          "metric": "tsd.hbase.puts",
          "tags": {
              "host": "tsdb-2.mysite.com"
          },
          "aggregatedTags": [],
          "dps": {
              "1365966001": 3902179270,
              "1365966062": 3902197769,
  ...
              "1365974281": 3922266478
          }
      }
  ]
