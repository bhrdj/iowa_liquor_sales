# Iowa Liquor Sales dataset via Socrata/data.iowa.gov

### (preliminary exploration)

The [state of Iowa has released an 800MB+ dataset of more than 3 million rows showing weekly liquor sales](https://data.iowa.gov/Economy/Iowa-Liquor-Sales/m3tr-qhgy), broken down by liquor category, vendor, and product name, e.g. `STRAIGHT BOURBON WHISKIES`, `Jim Beam Brands`, `Maker's Mark`

> This dataset contains the spirits purchase information of Iowa Class “E” liquor licensees by product and date of purchase from January 1, 2014 to current. The dataset can be used to analyze total spirits sales in Iowa of individual products at the store level.

You can view the dataset via [Socrata](https://data.iowa.gov/Economy/Iowa-Liquor-Sales/m3tr-qhgy)

Here are some steps to get the data wrangled. Do your own visualizations/analysis. But it looks like as good of dataset as any to see such things as:

- Purchase trends during holidays and college football season
- Most popular brands and types of alcohol
- Price variance between same-city stores and different-city stores
- Popularity of Hawkeye Vodka in Iowa City versus Ames


### Data caveats

Some visualizations and analyses have been done, and they strongly indicate that the data does not make for apples-to-apples comparisons.

via [Felipe Hoffa](https://twitter.com/felipehoffa) in [r/bigquery](http://www.reddit.com/r/bigquery/comments/37fcm6/iowa_liquor_sales_dataset_879mb_3million_rows/)

Notably, January 2015 has half the sales compared to January 2014. Since the dataset begins in Jan 2014...any number of things could be at play, such as a whole bunch of late-reported data being dumped into Jan. 2014. A store-by-store analysis is probably required to figure out the discrepancy. February sales also show a huge dip from 2014 to 2015. 

There's a substantial dip from May 2014 to June 2014, but I speculate that this is because Iowa's 3 major universities are out of session. However, sales from Aug. 2014 to Oct. 2014 don't show an appreciable buildup, even though school and football season restarts. In Dec. 2014, sales drop by more than half from November. Holiday trends/migration? Or another data collection oddity?

In short: doing time-series analysis is not recommended.






### The metadata

The URL for the metadata via Socrata's API, is:

https://data.iowa.gov/metadata/v1/dataset/m3tr-qhgy.json

Or you [can see a cached version here](https://gist.github.com/dannguyen/18ed71d3451d147af414#file-iowa-liquor-sales-metadata-json). The metadata contains column names and datatypes.


## Download it

~~~sh
# bash
curl https://data.iowa.gov/api/views/m3tr-qhgy/rows.csv?accessType=DOWNLOAD \
     -o iowa-liquor.csv
~~~


## Translate the dates, clean up numbers, pre-import

~~~sh
# via bash
sed -E "s#([0-9]{2})/([0-9]{2})/([0-9]{4})#\3-\1-\2#" < iowa-liquor.csv | 
  tr -d '$' > iowa-liquor-datefixed.csv
~~~


## SQL

Basic MySQL schema to include all the fields; however, you can probably drop the redundant `STORE LOCATION` field, at the very least.

~~~sql
# mysql
CREATE TABLE `iowaalcohol` (
  `DATE` date DEFAULT NULL,
  `CONVENIENCE STORE` varchar(255) DEFAULT NULL,
  `STORE` varchar(12) DEFAULT NULL,
  `NAME` varchar(255) DEFAULT NULL,
  `ADDRESS` varchar(255) DEFAULT NULL,
  `CITY` varchar(255) DEFAULT NULL,
  `ZIPCODE` varchar(20) DEFAULT NULL,
  `STORE LOCATION` varchar(255) DEFAULT NULL,
  `COUNTY NUMBER` varchar(4) DEFAULT NULL,
  `COUNTY` varchar(255) DEFAULT NULL,
  `CATEGORY` varchar(20) DEFAULT NULL,
  `CATEGORY NAME` varchar(100) DEFAULT NULL,
  `VENDOR NO` varchar(20) DEFAULT NULL,
  `VENDOR` varchar(255) DEFAULT NULL,
  `ITEM` varchar(20) DEFAULT NULL,
  `DESCRIPTION` varchar(255) DEFAULT NULL,
  `PACK` int(11) DEFAULT NULL,
  `LITER SIZE` int(11) DEFAULT NULL,
  `STATE BTL COST` float(7,2) DEFAULT NULL,
  `BTL PRICE` float(7,2) DEFAULT NULL,
  `BOTTLE QTY` int(11) DEFAULT NULL,
  `TOTAL` float(12,2) DEFAULT NULL,
  KEY `STORE` (`STORE`),
  KEY `DATE` (`DATE`),
  KEY `CATEGORY NAME` (`CATEGORY NAME`),
  KEY `CATEGORY` (`CATEGORY`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
~~~



## Dirty store names data

The store names aren't normalized; however, `STORE` seems like it should be a reliable enough foreign key for this other dataset on Socrata: [Iowa Liquor Stores](https://data.iowa.gov/Economy/Iowa-Liquor-Stores/ykb6-ywnd).

~~~sql
SELECT `STORE`, `NAME`, `ADDRESS`, `CITY`, `ZIPCODE`, `COUNTY NUMBER`, `COUNTY` 
FROM iowaalcohol
WHERE `STORE` = '2508'
GROUP BY `STORE`, `NAME`
~~~


##### Result

| STORE |                 NAME                |          ADDRESS          |     CITY     | ZIPCODE | COUNTY NUMBER | COUNTY |
|-------|-------------------------------------|---------------------------|--------------|---------|---------------|--------|
|  2508 | Hy-Vee Food Store #1 / Cedar Rapids | 1843 JOHNSON AVENUE, N.W. | CEDAR RAPIDS |   52405 |            57 | Linn   |
|  2508 | Hy-vee Food Store #1/ceda           | 1843 JOHNSON AVENUE N.W.  | CEDAR RAPIDS |   52405 |            57 | Linn   |
|  2508 | Hy-vee Food Store #1/Cedar Rapids   | 1843 JOHNSON AVENUE N.W.  | CEDAR RAPIDS |   52405 |            57 | Linn   |

## Top spirits by total sales

`CATEGORY NAME` is seemingly cleaner...here's how to get a quick summation of liquor categories, ordered by __total sales__, e.g. `SUM(TOTAL)`:

~~~sql
SELECT `CATEGORY NAME`,
ROUND(SUM(`TOTAL` / 1000)) as `total_sales`,
SUM(`BOTTLE QTY`) AS `total_bottles`,
ROUND(SUM(`LITER SIZE` * `BOTTLE QTY` / 1000) / 1000, 2) AS `total_liters`,
TRUNCATE(AVG(`TOTAL` / (`LITER SIZE` * `BOTTLE QTY` / 1000)), 2) AS `avg_cost_per_liter`
FROM iowaalcohol
GROUP BY `CATEGORY NAME`
ORDER BY `total_sales` DESC
~~~


Note: I don't really know if I'm interpreting the `BOTTLE QTY` and the seemingly irrelevant `PACK` columns correctly.


#### Top selling liquors in Iowa since January 2014

Note: `total_sales` and `total_liters` are in the _thousands_.

|           CATEGORY NAME            | total_sales | total_bottles | total_liters | avg_cost_per_liter |
|------------------------------------|-------------|---------------|--------------|--------------------|
| CANADIAN WHISKIES                  |       48053 |       3577933 |      3840.07 |              15.81 |
| 80 PROOF VODKA                     |       48046 |       5960351 |      5889.17 |               9.46 |
| SPICED RUM                         |       31601 |       2082680 |      2054.59 |              15.76 |
| IMPORTED VODKA                     |       23880 |       1166160 |      1138.00 |              25.07 |
| TEQUILA                            |       21411 |       1274034 |      1049.31 |              29.32 |
| STRAIGHT BOURBON WHISKIES          |       20924 |       1243488 |      1180.98 |              20.55 |
| WHISKEY LIQUEUR                    |       19339 |       1282480 |      1145.82 |              17.57 |
| TENNESSEE WHISKIES                 |       17648 |        804769 |       648.92 |              28.09 |
| PUERTO RICO & VIRGIN ISLANDS RUM   |       12729 |       1144599 |      1229.44 |              11.89 |
| BLENDED WHISKIES                   |       12037 |       1310974 |      1262.54 |              10.82 |
| FLAVORED VODKA                     |       11539 |       1124827 |       870.39 |              13.92 |
| MISC. IMPORTED CORDIALS & LIQUEURS |       11417 |        562464 |       437.35 |              28.72 |
| CREAM LIQUEURS                     |        9342 |        506558 |       422.24 |              22.25 |
| IMPORTED VODKA - MISC              |        9077 |        548380 |       402.24 |              23.72 |
| FLAVORED RUM                       |        8030 |        610725 |       532.19 |              15.16 |
| IMPORTED GRAPE BRANDIES            |        7742 |        465402 |       196.74 |              42.84 |
| SCOTCH WHISKIES                    |        7309 |        343235 |       387.97 |              26.27 |
| IMPORTED SCHNAPPS                  |        7076 |        410570 |       379.27 |              21.35 |
| AMERICAN COCKTAILS                 |        6314 |        602536 |       914.35 |               7.43 |
| IRISH WHISKIES                     |        5944 |        246198 |       209.89 |              31.13 |
| IMPORTED DRY GINS                  |        5391 |        237069 |       228.14 |              24.63 |
| AMERICAN DRY GINS                  |        5268 |        741783 |       580.66 |              10.12 |
| AMERICAN GRAPE BRANDIES            |        5137 |        854924 |       420.41 |              13.01 |
| DECANTERS & SPECIALTY PACKAGES     |        4449 |        234289 |       213.42 |              27.98 |
| SINGLE MALT SCOTCH                 |        4149 |         99707 |        76.89 |              57.69 |
| MISC. AMERICAN CORDIALS & LIQUEURS |        3759 |        297507 |       209.40 |              17.71 |
| STRAIGHT RYE WHISKIES              |        3755 |        142562 |       106.65 |              35.08 |
| COFFEE LIQUEURS                    |        2614 |        157633 |       131.57 |              19.54 |
| DISTILLED SPIRITS SPECIALTY        |        2601 |        256087 |       235.09 |              20.95 |
| PEACH SCHNAPPS                     |        1755 |        174310 |       159.22 |              10.95 |
| PEPPERMINT SCHNAPPS                |        1715 |        259103 |       235.03 |               8.34 |
| BLACKBERRY BRANDIES                |        1254 |        141897 |       123.74 |              10.47 |
| TRIPLE SEC                         |         986 |        253861 |       248.09 |               4.34 |
| AMERICAN AMARETTO                  |         885 |        138302 |       128.33 |               7.40 |
| AMERICAN ALCOHOL                   |         870 |         65116 |        49.09 |              17.78 |
| APPLE SCHNAPPS                     |         805 |         76803 |        68.15 |              12.21 |
| BUTTERSCOTCH SCHNAPPS              |         638 |         66228 |        56.66 |              11.13 |
| CINNAMON SCHNAPPS                  |         618 |         55962 |        46.66 |              15.31 |
| IMPORTED AMARETTO                  |         591 |         28728 |        21.32 |              27.79 |
| WATERMELON SCHNAPPS                |         502 |         45811 |        43.15 |              11.88 |
| MISCELLANEOUS SCHNAPPS             |         502 |         44459 |        37.38 |              14.66 |
| APRICOT BRANDIES                   |         501 |         56671 |        48.33 |              10.70 |
| GRAPE SCHNAPPS                     |         444 |         40053 |        38.42 |              11.83 |
| BARBADOS RUM                       |         396 |         27347 |        20.51 |              20.11 |
| JAMAICA RUM                        |         371 |         22730 |        18.78 |              20.18 |
| SINGLE BARREL BOURBON WHISKIES     |         356 |         12079 |         9.01 |              44.42 |
| 100 PROOF VODKA                    |         320 |         23346 |        18.87 |              17.13 |
| ROOT BEER SCHNAPPS                 |         268 |         29322 |        28.15 |              10.13 |
| PEACH BRANDIES                     |         215 |         31166 |        18.29 |              11.70 |
| FLAVORED GIN                       |         207 |         20939 |        15.10 |              14.02 |
| CHERRY BRANDIES                    |         201 |         25809 |        18.09 |              11.35 |
| RASPBERRY SCHNAPPS                 |         175 |         20777 |        17.93 |              10.26 |
| STRAWBERRY SCHNAPPS                |         166 |         22031 |        16.52 |              10.34 |
| TROPICAL FRUIT SCHNAPPS            |         124 |         17031 |        15.43 |               8.13 |
| MISCELLANEOUS BRANDIES             |         115 |          5344 |         3.60 |              38.19 |
| GREEN CREME DE MENTHE              |          95 |         13751 |        10.32 |               9.25 |
|                                    |          90 |          2903 |         2.18 |              40.02 |
| WHITE CREME DE CACAO               |          77 |         10958 |         8.22 |               9.30 |
| LOW PROOF VODKA                    |          76 |          6376 |         8.98 |              12.66 |
| DARK CREME DE CACAO                |          73 |         10383 |         7.78 |               9.30 |
| AMERICAN SLOE GINS                 |          70 |         10028 |         8.48 |               8.24 |
| OTHER PROOF VODKA                  |          57 |          4719 |         3.54 |              15.91 |
| BOTTLED IN BOND BOURBON            |          52 |          3603 |         2.93 |              19.01 |
| ROCK & RYE                         |          48 |          4566 |         3.42 |              14.54 |
| SPEARMINT SCHNAPPS                 |          41 |          5645 |         5.65 |               7.24 |
| WHITE CREME DE MENTHE              |          24 |          3440 |         2.58 |               9.35 |
| CREME DE ALMOND                    |          14 |          2007 |         1.52 |               9.30 |
| ANISETTE                           |          12 |          1697 |         1.27 |               9.26 |
| HIGH PROOF BEER                    |           4 |            38 |         0.03 |             145.56 |



