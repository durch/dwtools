# dwtools

**Current version: pre 1.0.0**  

Data Warehouse tools. Extension for `data.table` package for Data Warehouse related functionalities and some other helper functions.  
See below for core functions in the package.  
Report any bugs as issues on github.

## Installation

```r
devtools::install_github("jangorecki/dwtools")
```

## Core functions

```r
library(dwtools)
options("dwtools.verbose" = 0) # 1+ for status message
```

### dw.populate
Not core function but it will populate data for the next examples.  
`?dw.populate`

```r
X = dw.populate(scenario="star schema")
SALES = X$SALES
GEOGRAPHY = X$GEOGRAPHY
head(SALES)
#>    cust_code prod_code geog_code  time_code curr_code   amount
#> 1:     id088       649        UT 2010-01-02       CZK 883.8430
#> 2:     id097       139        NM 2011-09-08       IDR 507.5106
#> 3:     id087       527        NY 2010-01-10       IDR 423.3552
#> 4:     id044       702        OR 2013-02-21       FTC 304.5715
#> 5:     id020        78        NJ 2013-10-25       XDG 973.9260
#> 6:     id009       339        WI 2011-04-19       XVN 817.0486
#>           value
#> 1:     92.28056
#> 2: 191600.92622
#> 3: 803561.87327
#> 4: 435609.55580
#> 5: 998722.79516
#> 6: 850158.35636
head(GEOGRAPHY)
#>    geog_code  geog_name geog_division_code geog_division_name
#> 1:        AK     Alaska                  9            Pacific
#> 2:        AL    Alabama                  4 East South Central
#> 3:        AR   Arkansas                  5 West South Central
#> 4:        AZ    Arizona                  8           Mountain
#> 5:        CA California                  9            Pacific
#> 6:        CO   Colorado                  8           Mountain
#>    geog_region_code geog_region_name
#> 1:                4             West
#> 2:                2            South
#> 3:                2            South
#> 4:                4             West
#> 5:                4             West
#> 6:                4             West
```

### db
Function provides simple database interface.  
It handles DBI drivers (tested on Postgres and SQLite), RODBC (any odbc connection, not yet tested) and csv files as tables.  
NoSQL couchdb support in dev.  
In ETL terms where `data.table` serves as **Transformation** layer, the dwtools `db` function serves **Extraction** and **Loading** layers.  
`?db`

```r
# setup db connections
library(RSQLite) # install.packages("RSQLite")
#> Loading required package: DBI
sqlite1 = list(drvName="SQLite",dbname="sqlite1.db",conn=dbConnect(SQLite(), dbname="sqlite1.db"))
options("dwtools.db.conns" = list(sqlite1=sqlite1, csv1=list(drvName="csv")))

# write to db (default connection)
db(SALES,"sales_table")
# read from from db
db("sales_table")
# query from db # not supported for csv driver
db("SELECT * FROM sales_table")
# send to db # not supported for csv driver
db("DROP TABLE sales_table")

# write geography data.table into multiple connections
db(GEOGRAPHY,"geography",c("sqlite1","csv1"))
# lookup from db and setkey
db("geography",key="geog_code")
# use data.table chaining
SALES[,.(amount=sum(amount),value=sum(value)),keyby=list(geog_code,time_code) # aggr to geog_code, time_code
      ][,db("geography",key="geog_code")[.SD] # lookup from sqlite1
        ][,.(amount=sum(amount),value=sum(value)),keyby=list(geog_division_name,time_code) # aggr to division_code, time_code
          ]
SALES[,.SD,keyby=list(geog_code) # setkey
      ][db("geography","csv1",key="geog_code" # join to geography from csv file
           )[geog_division_name=="East South Central" # filter geography to one division_name
             ], nomatch=0 # inner join
        ]
```
`db` function accepts vector of sql statements / table names to allow batch processing.  
In case of tables migration see `?dbCopy`.

### joinbyv
Batch join multiple tables into one master table.  
Denormalization of star schema and snowflake schema to flat fact table.  
`?joinbyv`

```r
names(X) # list tables in star schema
#> [1] "SALES"     "CUSTOMER"  "PRODUCT"   "GEOGRAPHY" "TIME"      "CURRENCY"
sapply(X, nrow) # nrow of each tbl
#>     SALES  CUSTOMER   PRODUCT GEOGRAPHY      TIME  CURRENCY 
#>    100000       100      1000        50      1826        49
# denormalize 
DT = joinbyv(
  master = X$SALES,
  join = list(customer = X$CUSTOMER,
              product = X$PRODUCT,
              geography = X$GEOGRAPHY,
              time = X$TIME,
              currency = X$CURRENCY),
  col.subset = list(c("cust_active"),
                    c("prod_group_name","prod_family_name"),
                    c("geog_region_name"),
                    c("time_month_name"),
                    NULL)
  )
print(names(DT))
#>  [1] "curr_code"        "currency_type"    "time_month_name" 
#>  [4] "geog_region_name" "prod_group_name"  "prod_family_name"
#>  [7] "cust_active"      "cust_code"        "prod_code"       
#> [10] "geog_code"        "time_code"        "amount"          
#> [13] "value"
```

### idxv
Also known as *Nth setkey*.  
Creates custom indices for a data.table object. May require lot of memory.  

```r
DT = dw.populate(scenario="fact")
# custom indices
Idx = list(
  c("cust_code", "geog_code", "curr_code"),
  c(2:3)
)
IDX = idxv(DT, Idx)

# binary search using first index, equivalent to: DT[cust_code=="id006" & geog_code=="UT" & curr_code=="NOK"]
DT[CJI(IDX,"id006",TRUE,"UT",TRUE,"NOK")]
# binary search using second index, equivalent to: DT[prod_code==323 & geog_code=="OR"]
DT[CJI(IDX,TRUE,323,"OR")]
```

## Other functions
A brief summary of other functions in the package.  
* `timing` - measure time, nrow in/out, optional save to db
* `as.xts` - wrapper method for conversion of data.table to xts
* `vwap` - aggregate tick trades data to OHLC including VWAP

## License
GPL-3.  
Donations are welcome and will be partially forwarded to dependencies of dwtools.  
[19JRajumtMNU9h9Wvdpsnq13SRdZjfbLeN](https://blockchain.info/address/19JRajumtMNU9h9Wvdpsnq13SRdZjfbLeN)

## Contact
`J.Gorecki@wit.edu.pl`

