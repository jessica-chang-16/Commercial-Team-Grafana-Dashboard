> Count of Crawlers per Region
```
WITH last_retailer_timestamp AS 
  (SELECT max(timestamp) as timestamp, 
  retailer_id 
  from clickhouse_data_table_01 
  GROUP BY retailer_id
)
SELECT region, uniq(retailer_id) AS crawlers_per_region
FROM clickhouse_data_table_01  FINAL
INNER JOIN last_retailer_timestamp USING (timestamp, retailer)
WHERE $__timeFilter(timestamp)
GROUP BY region
ORDER BY crawlers_per_region DESC;
```

> Crawlers by Region
```
WITH last_retailer_timestamp AS (
  SELECT max(timestamp) as timestamp, retailer_id from clickhouse_data_table_01 AS Products_Tracked GROUP BY retailer_id
)
SELECT splitByChar('-', retailer_id)[1] AS "retailer name", region, retailer AS nickname
FROM clickhouse_data_table_01 AS Products_Tracked FINAL
INNER JOIN last_retailer_timestamp USING (timestamp, retailer_id)
WHERE $__timeFilter(timestamp)
GROUP BY retailer_id, region
ORDER BY retailer_id, region ASC
```

> Crawler Count by Retailer
```
WITH last_retailer_timestamp AS
(
  SELECT max(timestamp) as timestamp,
  retailer_id from clickhouse_data_table_01
  GROUP BY retailer_id
)
SELECT splitByChar('-', retailer)[1] as retailer_no_region, 
COUNT(region_id) as total_count_regions,
groupArray(region_id) as regions
FROM clickhouse_data_table_01 FINAL
INNER JOIN last_retailer_timestamp USING (timestamp, retailer_id)
WHERE $__timeFilter(timestamp)
GROUP BY retailer_no_region
ORDER BY retailer_no_region ASC
```

> Average Staleness > 5 days
```
SELECT concat(retailer_id, ' [', number_rank, ']') as nickname, (avgWeighted(product_staleness_column1, product_staleness_count_column2)) / 86400 as product_staleness
FROM clickhouse_data_table_02 FINAL
WHERE $__timeFilter(timestamp)
AND $__conditionalAll(retailer_id IN (${Retailer_id:sqlstring}), $Retailer_id)
GROUP BY retailer_id, number_rank
ORDER BY product_staleness DESC
LIMIT 5
```

> Average Percent of Products Scraped
```
SELECT retailer_id,
        number_rank,
        avg(
          if(
        isNaN(
            products_scraped_column1 / 
            (products_scraped_column2y + COALESCE(products_errored_column1, 0)) * 100
        ),
        0,
        products_scraped_column1 / 
        (products_scraped_column2 + COALESCE(products_errored_column1, 0)) * 100
    )
    ) AS products_scraped_pct
FROM clickhouse_data_table_02 FINAL
WHERE 
location = 'location1'
AND
$__timeFilter(timestamp) 
AND
$__conditionalAll(retailer_id IN (${retailer_id:sqlstring}), $retailer_id)
GROUP BY retailer_id, number_rank
ORDER BY "products_scraped_pct" ASC, retailer_id DESC
LIMIT 10
```

> Percent of Products Scraped Daily
```
WITH filtered_retailers_id AS
  (SELECT retailer_id,
  FROM clickhouse_data_table_02 FINAL
  WHERE (location = 'location1') 
  AND $__timeFilter(timestamp)
  AND $__conditionalAll(retailer_id IN (${retailer_id:sqlstring}), $retailer_id)
  GROUP BY retailer_id
  LIMIT 5
  )
SELECT retailer_id,
        toDate(timestamp + INTERVAL 1 Day) AS time,
       (case when products_scraped_column2 > 0 
       then products_scraped_column1 / (products_scraped_5d_column2 + COALESCE(products_errored_column1,0))  
       else 0 END * 100 ) AS "products_scraped_pct"
FROM clickhouse_data_table_02 FINAL
WHERE 
  retailer_id IN (SELECT retailer_id from filtered_retailers)
ORDER BY timestamp, "products_scraped_pct", retailer_id
```
