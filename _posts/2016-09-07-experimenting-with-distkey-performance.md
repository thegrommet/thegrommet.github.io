Redshift &mdash; Amazon's enterprise-level, columnar data warehousing service &mdash; distributes data to node slices within a cluster according to one of three distribution styles: key, all, or even. 

With key distribution, rows with the same values in a specified distkey column are placed in the same node. An all distribution style, in turn, copies the entire table to each node in the cluster. Finally, even distribution, the default distribution style, allocates table rows to the nodes in a sequential or round-robin manner.

Choosing the right distribution approach is critical to query performance. Indeed, by placing data where it needs to be prior to query execution, an optimal data distribution strategy may not only reduce the amount of physical movement of data, but also minimize discrepancies in the amount of work done by the nodes within the cluster. 

Each of the abovementioned distribution styles can (and likely should) perform a role in an effective, comprehensive, and planned data warehouse design approach; however, particularly careful consideration and consequence should be given to utilizing a distkey style of distribution on a given table. While the distkey can afford certain efficiencies around joins and aggregations, it's surprisingly easy to implement poorly and in such a way that query execution is actually hindered. 

In an effort to explore some of the nuances around distkey usage, we conducted an informal, comparison-focused experiment looking at the impact of the distkey on general query performance.

<strong>BENCHMARKING THE DISTKEY</strong>

We began by creating two tables composed of two columns each: one column to hold a unique record id and one to hold an event_code (e.g., an integer representing some hypothetical transaction or interaction). We gave one of the tables a distkey on the event_code column.

{% highlight shell %}
#dist key of event_code
create table bits_dist(event_id bigint identity(0,1), event_code smallint, primary key(event_id)) distkey(event_code);

#no distkey
create table bits_nodist(event_id bigint identity(0,1), event_code smallint, primary key(event_id));
{% endhighlight %}

We populated each of the tables with 1 million rows of data, with each event_code record for each table consisting of a matching randomly generated integer between 1 and 5. We ran the following query 1000 times using PDO methods on an AWS EC2 instance:

{% highlight shell %}
#distkey query
select event_code, count(*) from bits_dist group by event_code;

#no distkey query
select event_code, count(*) from bits_nodist group by event_code;
{% endhighlight %}

Calculating the average time required for each query to run, we received the following results:

<table style="width:100%;">
<tr>
<th colspan="2">1 million rows</th>
</tr>
<tr>
<th>DISTKEY (in seconds)</th>
<th>NO DISTKEY (in seconds)</th>
</tr>
<tr>
<td>0.1172</td>
<td>0.1449</td>
</tr>
</table>

As shown, the table with the distkey executed the query faster than the table without a distkey. This outcome generally adheres to what one might expect: that is, because a distkey guarantees that all matching values in the distkey column will be on the same node, our distkey table likely required less data redistribution (and, by extension, less time) during query execution than the table without a distkey.

We next increased the number of rows in each table to 10 million and ran the same queries again:

<table style="width:100%;">
<tr>
<th colspan="2">10 million rows</th>
</tr>
<tr>
<th>DISTKEY (in seconds)</th>
<th>NO DISTKEY (in seconds)</th>
</tr>
<tr>
<td>0.2948</td>
<td>0.2242</td>
</tr>
</table>

Unexpectedly, with an increase in the number of records, the table without a distkey actually performed somewhat better than the table with a distkey. If our suggestion above about the distkey’s impact on redistribution held, shouldn’t more rows result in more time for the table with an even (e.g. default) distribution style?

To obtain more data, we benchmarked the same set of queries twice more against each table, once with 20 million rows and again with 30 million rows.

<table style="width:100%;margin-bottom:20px;">
<tr>
<th colspan="2">20 million rows</th>
</tr>
<tr>
<th>DISTKEY (in seconds)</th>
<th>NO DISTKEY (in seconds)</th>
</tr>
<tr>
<td>0.5151</td>
<td>0.3098</td>
</tr>
</table>

<table style="width:100%;">
<tr>
<th colspan="2">30 million rows</th>
</tr>
<tr>
<th>DISTKEY (in seconds)</th>
<th>NO DISTKEY (in seconds)</th>
</tr>
<tr>
<td>0.8102</td>
<td>0.4517</td>
</tr>
</table>

In both instances, executing the query on the table with a distkey continued to be slower than executing that same query on the table without a distkey. Why?

<strong>TABLE PERFORMANCE</strong>

One of the most well-documented reasons for poor distkey results in Redshift is data skew. Essentially, if a distkey column has an imbalanced distribution of values, some nodes will receive a greater number of records than other nodes. This, in turn, forces those nodes with more records to carry a greater portion of the query workload, negatively impacting overall query performance.

To determine whether or not our experimental dataset was skewed, we performed a basic count on each table, with the following results:

<table style="width:100%;">
<tr>
<th>EVENT_CODE</th>
<th>COUNT</th>
</tr>
<tr>
<td>1</td>
<td>5997503</td>
</tr>
<tr>
<td>2</td>
<td>6001146</td>
</tr>
<tr>
<td>3</td>
<td>6001590</td>
</tr>
<tr>
<td>4</td>
<td>5999052</td>
</tr>
<tr>
<td>5</td>
<td>6000709</td>
</tr>
</table>

While there was some variation in our dataset, it was minor, with the difference between the largest count and the smallest count coming in at just over 4000 rows. Since data skew alone was likely not the primary cause of the distkey table’s deficient performance, we had to investigate further.

Buried in the AWS documentation, there is an especially useful query that provides a high-level overview of table design across the cluster:

{% highlight shell %}
SELECT SCHEMA schemaname,"table" tablename,table_id tableid,size size_in_mb,
CASE
WHEN diststyle NOT IN ('EVEN','ALL') THEN 1
ELSE 0
END has_dist_key,
CASE
WHEN sortkey1 IS NOT NULL THEN 1
ELSE 0
END has_sort_key,
CASE
WHEN encoded = 'Y' THEN 1
 ELSE 0
END has_col_encoding,
CAST(max_blocks_per_slice - min_blocks_per_slice AS FLOAT) / GREATEST(NVL (min_blocks_per_slice,0)::int,1) ratio_skew_across_slices,
CAST(100*dist_slice AS FLOAT) /(SELECT COUNT(DISTINCT slice) FROM stv_slices) pct_slices_populated
FROM svv_table_info ti
JOIN (SELECT tbl,MIN(c) min_blocks_per_slice,MAX(c) max_blocks_per_slice, COUNT(DISTINCT slice) dist_slice
FROM (SELECT b.tbl,b.slice,COUNT(*) AS c
FROM STV_BLOCKLIST b
GROUP BY b.tbl, b.slice)
WHERE tbl IN (SELECT table_id FROM svv_table_info)
GROUP BY tbl) iq ON iq.tbl = ti.table_id;
{% endhighlight %}

Running this query returned some helpful and interesting results:

<table>
<tr>
<th>tablename</th>
<th>has_dist_key</th>
<th>ratio_skew_across_slices</th>
<th>pct_slices_populated</th>
</tr>
<tr>
<td>bits_nodist</td>
<td>0</td>
<td>0</td>
<td>100</td>
</tr>
<tr>
<td>bits_dist</td>
<td>1</td>
<td>1.96</td>
<td>50</td>
</tr>
</table>
<span style="display:block;text-align:center;font-size:10px;font-style:italic;">(some columns intentionally omitted)</span>

The column "ratio_skew_across_slices" shows the distribution skew for the table (where a smaller number is ideal), while "pct_slices_populated" indicates the percentage of total slices populated by data from the table (where a larger number is ideal). The table without a distkey, as shown, has no skew and populates 100% of the slices. This is what we would expect from an even distribution style. The table with a distkey, however, not only suffers from skew, but also, and more importantly, populates only half of the available slices across the cluster.

<strong>CONCLUSIONS</strong>

The distkey we chose lacked one of the quintessential properties of an ideal distkey column: high cardinality. This is to say that in order to get optimal results from implementing a particular column as a distkey, that column needs not only a balanced distribution of data within the column, but also enough unique values so that all nodes in the cluster are utilized.

In our case, we had only five unique values in our event_code column. So, even though the data itself did not suffer from significant skew, the low cardinality of the column dataset lead to an imbalanced distribution of that data to the nodes. As a result, the table without a distkey &mdash; which suffered from a redistribution problem, but utilized all of the nodes &mdash; was generally more performant than the distkey table’s partial usage of the cluster’s total computing capacity.
