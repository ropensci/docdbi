
# nodbi

[![CRAN](https://cranchecks.info/badges/worst/nodbi)](https://cran.r-project.org/package=nodbi)
[![Project Status: Active – The project has reached a stable, usable
state and is being actively
developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![cran
checks](https://cranchecks.info/badges/worst/nodbi)](https://cranchecks.info/pkgs/nodbi)
[![R-CMD-check](https://github.com/ropensci/nodbi/workflows/R-CMD-check/badge.svg)](https://github.com/ropensci/nodbi/actions?query=workflow%3AR-CMD-check)
[![codecov](https://codecov.io/gh/rfhb/nodbi/branch/master/graph/badge.svg)](https://app.codecov.io/gh/rfhb/nodbi)
[![rstudio mirror
downloads](http://cranlogs.r-pkg.org/badges/nodbi)](https://github.com/r-hub/cranlogs.app)
[![cran
version](https://www.r-pkg.org/badges/version/nodbi)](https://cran.r-project.org/package=nodbi)

`nodbi` is an R package that provides a single interface for several
NoSQL databases and databases with JSON functionality, with the same
function parameters and return values across all database backends. Last
updated 2023-07-28.

| Currently, `nodbi` supports as database backends | for an `R` object of any of these data types | for the operations |
|--------------------------------------------------|----------------------------------------------|--------------------|
| MongoDB                                          | data.frame                                   | List, Exists       |
| SQLite                                           | list                                         | Create             |
| PostgreSQL                                       | JSON string                                  | Get                |
| DuckDB                                           | file name of NDJSON records                  | Query\*            |
| Elasticsearch                                    | URL of NDJSON records                        | Update\*           |
| CouchDB                                          |                                              | Delete             |

Limitations: \*For Elasticsearch, only simple queries (e.g. equality for
a single field) and for CouchDB only root fields.

For a speed comparisons of database backends, see
[benchmark](#benchmark) below.

## API overview

Parameters for `docdb_*()` functions are the same across all database
backends. See [walk-through](#walk-through) below and the canonical
testing in [core-nodbi.R](./tests/testthat/core-nodbi.R). “Container” is
used as term to indicate where conceptually the backend holds the data,
see [Database connections](#database-connections) below. The `key`
parameter holds the name of a container.

| Purpose                                                                                                                                 | Function call                                                                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Create database connection (see below)                                                                                                  | `src <- nodbi::src_{duckdb, postgres, mongo, sqlite, couchdb, elastic}(<see below for parameters>)`                      |
| Load `my_data` (a data frame, a list, a JSON string, or a file name or url with NDJSON records) into database, container `my_container` | `nodbi::docdb_create(src = src, key = "my_container", value = my_data)`                                                  |
| Get all documents back into a data frame                                                                                                | `nodbi::docdb_get(src = src, key = "my_container")`                                                                      |
| Get documents selected with query (as MongoDB-compatible JSON) into a data frame                                                        | `nodbi::docdb_query(src = src, key = "my_container", query = '{"age": 20}')`                                             |
| Get selected fields (in MongoDB compatible JSON) from documents selected query                                                          | `nodbi::docdb_query(src = src, key = "my_container", query = '{"age": 20}', fields = '{"name": 1, "_id": 0, "age": 1}')` |
| Update (patch) selected documents with new data in data frame, list or JSON string from `my_data`                                       | `nodbi::docdb_update(src = src, key = "my_container", value = my_data, query = '{"age": 20}')`                           |
| Check if container exists                                                                                                               | `nodbi::docdb_exists(src = src, key = "my_container")`                                                                   |
| List all containers in database                                                                                                         | `nodbi::docdb_list(src = src)`                                                                                           |
| Delete document(s) in container                                                                                                         | `nodbi::docdb_delete(src = src, key = "my_container", query = '{"age": 20}')`                                            |
| Delete container                                                                                                                        | `nodbi::docdb_delete(src = src, key = "my_container")`                                                                   |
| Close and remove database connection manually (when restarting R, connections are closed and removed automatically by `nodbi`)          | `rm(src)`                                                                                                                |

## Install

CRAN version

``` r
install.packages("nodbi")
```

Development version

``` r
remotes::install_github("ropensci/nodbi")
```

Load package from library

``` r
library("nodbi")
```

## Database connections

Overview on parameters and aspects that are specific to the database
backend. These are only needed once: for `src_*()` to create a
connection object. The connection object is subsequently used with
`docdb_*` functions. “Container” refers to where conceptually the
backend holds the data. Data types are mapped from JSON to R objects by
[jsonlite](https://CRAN.R-project.org/package=jsonlite).

### DuckDB

See also <https://CRAN.R-project.org/package=duckdb>. “Container” refers
to a DuckDB table, with columns `_id` and `json` created and used by
package `nodbi`, applying SQL functions as per
<https://duckdb.org/docs/extensions/json> to the `json` column. Each row
in the table represents a `JSON` document. Any root-level `_id` is
extracted from the document(s) and used for column `_id`, otherwise a
UUID is created as `_id`. The table is indexed on `_id`.

``` r
src <- nodbi::src_duckdb(dbdir = ":memory:", ...)
```

### MongoDB

“Container” refers to a MongoDB collection, in which `nodbi` creates
JSON documents. If documents do not have root-level `_id`’s, UUID’s are
created as `_id`’s. MongoDB but none of the other databases require to
specify the container already in the `src_*()` function. See also
<https://jeroen.github.io/mongolite/>.

``` r
src <- nodbi::src_mongo(
  collection = "my_container", db = "my_database",
  url = "mongodb://localhost", ...)
```

### SQLite

“Container” refers to an SQLite table, with columns `_id` and `json`
created and used by package `nodbi`, applying SQL functions as per
<https://www.sqlite.org/json1.html> to the `json` column. Each row in
the table represents a `JSON` document. Any root-level `_id` is
extracted from the document(s) and used for column `_id`, otherwise a
UUID is created as `_id`. The table is indexed on `_id`. See also
<https://CRAN.R-project.org/package=RSQLite>.

``` r
src <- nodbi::src_sqlite(dbname = ":memory:", ...)
```

### CouchDB

“Container” refers to a CouchDB database, in which `nodbi` creates JSON
documents. If documents do not have root-level `_id`’s, UUID’s are
created as `_id`’s. With CouchDB, function `docdb_update()` uses
[jqr](https://cran.r-project.org/package=jqr) to implement patching
JSON, in analogy to functions available for the other databases. See
also <https://CRAN.R-project.org/package=sofa>.

``` r
src <- nodbi::src_couchdb(
  host = "127.0.0.1", port = 5984L, path = NULL,
  transport = "http", user = NULL, pwd = NULL, headers = NULL)
```

### Elasticsearch

“Container” refers to an Elasticsearch index, in which `nodbi` creates
JSON documents. Any root-level `_id` is extracted from the document(s)
and used as document ID `_id`, otherwise a UUID is created as document
ID `_id`. Only lowercase is accepted for container names (in parameter
`key`). Opensearch can equally be used. See also
<https://CRAN.R-project.org/package=elastic>.

``` r
src <- nodbi::src_elastic(
  host = "127.0.0.1", port = 9200L, path = NULL,
  transport_schema = "http", user = NULL, pwd = NULL, force = FALSE, ...)
```

### PostgreSQL

“Container” refers to a PostgreSQL table, with columns `_id` and `json`
created and used by package `nodbi`, applying SQL functions as per
<https://www.postgresql.org/docs/current/functions-json.html> to the
`json` column. Each row in the table represents a `JSON` document. Any
root-level `_id` is extracted from the document(s) and used for column
`_id`, otherwise a UUID is created as `_id`. The table is indexed on
`_id`. With PostgreSQL, a custom `plpgsql` function
[jsonb_merge_patch()](https://github.com/ropensci/nodbi/blob/master/R/src_postgres.R#L60)
is used for `docdb_update()`. The order of variables in data frames
returned by `docdb_get()` and `docdb_query()` can differ from their
order the input to `docdb_create()`. See also
<https://CRAN.R-project.org/package=RPostgreSQL>.

``` r
src <- nodbi::src_postgres(
  dbname = "my_database", host = "127.0.0.1", port = 5432L, ...)
```

## Walk-through

This example is meant to show how functional `nodbi` is at this time.

``` r
# load nodbi
library(nodbi)

# name of container
key <- "my_container"

# connect database backend
# 
# parts of the example do not yet work with 
# these database backends, see *notes* below
src <- src_elastic()
src <- src_couchdb()
#
# this example fully works with any of
src <- src_mongo(collection = key)
src <- src_postgres()
src <- src_sqlite()
src <- src_duckdb()

# check if container already exists
docdb_exists(src, key)
# [1] FALSE

# load data (here data frame, alternatively list or JSON)
# into the container "my_container" specified in "key" parameter
docdb_create(src, key, value = mtcars)
# [1] 32

# load additionally 98 NDJSON records
docdb_create(src, key, "https://httpbin.org/stream/98")
# Note: container 'my_container' already exists
# [1] 98

# load additionally contacts JSON data, from package nodbi
docdb_create(src, key, contacts)
# Note: container 'my_container' already exists
# [1] 5

# get all documents, irrespective of schema
dplyr::tibble(docdb_get(src, key))
# # A tibble: 135 × 27
#    `_id`      url   args  headers$Host origin    id isActive balance   age eyeColor name  email
#    <chr>      <chr> <df[> <chr>        <chr>  <int> <lgl>    <chr>   <int> <chr>    <chr> <chr>
#  1 006c91d9-… http…       httpbin.org  83.13…    87 NA       NA         NA NA       NA    NA   
#  2 00bfd508-… http…       httpbin.org  83.13…    17 NA       NA         NA NA       NA    NA   
#  3 04f17695-… http…       httpbin.org  83.13…    53 NA       NA         NA NA       NA    NA   
#  4 05d0f2be-… http…       httpbin.org  83.13…    80 NA       NA         NA NA       NA    NA   
#  5 06b6b86d-… http…       httpbin.org  83.13…    55 NA       NA         NA NA       NA    NA   
#  6 0760c273-… http…       httpbin.org  83.13…    11 NA       NA         NA NA       NA    NA   
#  7 08de6593-… http…       httpbin.org  83.13…    47 NA       NA         NA NA       NA    NA   
#  8 0b1cb164-… http…       httpbin.org  83.13…    63 NA       NA         NA NA       NA    NA   
#  9 0bafc40e-… http…       httpbin.org  83.13…    60 NA       NA         NA NA       NA    NA   
# 10 11ba908b-… http…       httpbin.org  83.13…    74 NA       NA         NA NA       NA    NA   
# # ℹ 125 more rows
# # ℹ 18 more variables: headers$`X-Amzn-Trace-Id` <chr>, $`User-Agent` <chr>, $Accept <chr>,
# #   about <chr>, registered <chr>, tags <list>, friends <list>, mpg <dbl>, cyl <int>,
# #   disp <dbl>, hp <int>, drat <dbl>, wt <dbl>, qsec <dbl>, vs <int>, am <int>, gear <int>,
# #   carb <int>


# query some documents
# *note*: such complex queries do not yet work with src_elasticsearch()
docdb_query(src, key, query = '{"mpg": {"$gte": 30}}')
#              _id mpg cyl disp  hp drat  wt qsec vs am gear carb
# 1       Fiat 128  32   4   79  66  4.1 2.2   19  1  1    4    1
# 2    Honda Civic  30   4   76  52  4.9 1.6   19  1  1    4    2
# 3 Toyota Corolla  34   4   71  65  4.2 1.8   20  1  1    4    1
# 4   Lotus Europa  30   4   95 113  3.8 1.5   17  1  1    5    2

# query some fields from some documents; 'query' is a mandatory 
# parameter and is used here in its position in the signature
# *note*: such complex queries do not yet work with src_elasticsearch()
docdb_query(src, key, '{"mpg": {"$gte": 30}}', fields = '{"wt": 1, "mpg": 1}')
#    wt mpg
# 1 2.2  32
# 2 1.6  30
# 3 1.8  34
# 4 1.5  30

# query some subitem fields from some documents
# *note*: such complex queries do not yet work with
#         src_couchdb() or src_elasticsearch()
str(docdb_query(
  src, key, 
  query = '{"$or": [{"age": {"$gt": 21}}, 
           {"friends.name": {"$regex": "^B[a-z]{3,9}.*"}}]}', 
  fields = '{"age": 1, "friends.name": 1}'))
# 'data.frame': 3 obs. of  2 variables:
#  $ age    : int  23 30 22
#  $ friends:'data.frame':  3 obs. of  1 variable:
#   ..$ name:List of 3
#   .. ..$ : chr  "Wooten Goodwin" "Brandie Woodward" "Angelique Britt"
#   .. ..$ : chr  "Coleen Dunn" "Doris Phillips" "Concetta Turner"
#   .. ..$ : chr  "Baird Keller" "Francesca Reese" "Dona Bartlett"

# such queries can also be used for updating (patching) selected documents 
# with a new 'value'(s) from a JSON string, a data frame or a list
docdb_update(src, key, value = '{"vs": 9, "xy": [1, 2]}', query = '{"carb": 3}')
# [1] 3
# *note*: such queries do not yet work with src_elasticsearch()
docdb_query(src, key, '{"carb": {"$in": [1,3]}}', fields = '{"vs": 1}')[[1]]
# [1] 1 1 1 9 9 9 1 1 1 1
docdb_get(src, key)[c(43, 93, 100, 101), c("_id", "xy", "url", "email")]
#                                      _id   xy                           url                  email
# 43              5cd6785325ce3a94dfc54096 NULL                          <NA> pacebell@conjurica.com
# 93                            Merc 450SL 1, 2                          <NA>                   <NA>
# 100                           Volvo 142E NULL                          <NA>                   <NA>
# 101 a12f9901-4edf-4cdd-b3e6-3a9a7ca48c9d NULL https://httpbin.org/stream/98                   <NA>

# use with dplyr
# *note* that dplyr includes a (deprecated) function src_sqlite
# which would mask nodbi's src_sqlite, so it is excluded here
library("dplyr", exclude = c("src_sqlite", "src_postgres"))
# 
docdb_get(src, key) %>%
  group_by(gear) %>%
  summarise(mean_mpg = mean(mpg))
# # A tibble: 4 × 2
#    gear mean_mpg
#   <int>    <dbl>
# 1     3     16.1
# 2     4     24.5
# 3     5     21.4
# 4    NA     NA 

# delete documents; query is optional parameter and has to be 
# specified for deleting documents instead of deleting the container
# *note*: such complex queries do not yet work with src_couchdb() or src_elasticsearch()
dim(docdb_query(src, key, query = '{"$or": [{"age": {"$lte": 20}}, {"age": {"$gte": 25}}]}'))
# [1] 3 11
docdb_delete(src, key, query = '{"$or": [{"age": {"$lte": 20}}, {"age": {"$gte": 25}}]}')
# TRUE
nrow(docdb_get(src, key))
# [1] 132

# delete container from database
docdb_delete(src, key)
# [1] TRUE
# 
# shutdown
DBI::dbDisconnect(src$con, shutdown = TRUE); rm(src)
```

## Benchmark

``` r
library("nodbi")

srcMongo <- src_mongo()
srcSqlite <- src_sqlite()
srcPostgres <- src_postgres()
srcDuckdb <- src_duckdb()
srcElastic <- src_elastic()
srcCouchdb <- src_couchdb(
  user = Sys.getenv("COUCHDB_TEST_USER"), 
  pwd = Sys.getenv("COUCHDB_TEST_PWD"))

key <- "test"
query <- '{"clarity": "SI1"}'
fields <- '{"cut": 1, "_id": 1, "clarity": "1"}'
value <- '{"clarity": "XYZ", "new": ["ABC", "DEF"]}'
data <- as.data.frame(diamonds)[1:2000, ]
ndjs <- tempfile()
jsonlite::stream_out(iris, con = file(ndjs), verbose = FALSE)

testFunction <- function(src, key, value, query, fields) {
  on.exit(docdb_delete(src, key))
  suppressMessages(docdb_create(src, key, data))
  suppressMessages(docdb_create(src, key, ndjs))
  # Elasticsearch needs a delay to process the data
  if (inherits(src, "src_elastic")) Sys.sleep(1)
  head(docdb_get(src, key))
  docdb_query(src, key, query = query, fields = fields)
  docdb_update(src, key, value = value, query = query)
}

# 2023-07-28 with 2015 mobile hardware, databases via homebrew
rbenchmark::benchmark(
  MongoDB = testFunction(src = srcMongo, key, value, query, fields),
  SQLite = testFunction(src = srcSqlite, key, value, query, fields),
  Elastic = testFunction(src = srcElastic, key, value, query, fields),
  CouchDB = testFunction(src = srcCouchdb, key, value, query, fields),
  PostgreSQL = testFunction(src = srcPostgres, key, value, query, fields),
  DuckDB = testFunction(src = srcDuckdb, key, value, query, fields),
  replications = 10L,
  columns = c('test', 'replications', 'elapsed')
)
#         test replications elapsed
# 4    CouchDB           10   247.0
# 6     DuckDB           10     3.4
# 3    Elastic           10    50.7 # 10s to be subtracted
# 1    MongoDB           10     3.1
# 5 PostgreSQL           10     3.9
# 2     SQLite           10     3.3
```

## Testing

Every database backend is subjected to identical tests, see
[core-nodbi.R](./tests/testthat/core-nodbi.R).

``` r
# 2023-07-28
testthat::test_local()
# ✔ | F W S  OK | Context
# ✔ |     1  95 | couchdb [90.6s]                                                                 
# ✔ |       121 | duckdb [5.3s]                                                                   
# ✔ |     2  66 | elastic [82.7s]                                                                 
# ✔ |       120 | mongodb [5.9s]                                                                  
# ✔ |       124 | postgres [36.9s]                                                                
# ✔ |       123 | sqlite [36.0s]                                                                  
# 
# ══ Results ═════════════════════════════════════════════════════════════════════════════════════
# Duration: 257.6 s
# 
# ── Skipped tests (3) ───────────────────────────────────────────────────────────────────────────
# • bulk updates not yet implemented (2): core-nodbi.R:248:3, core-nodbi.R:248:3
# • queries need to be translated into elastic syntax (1): core-nodbi.R:180:3
# 
# [ FAIL 0 | WARN 0 | SKIP 3 | PASS 649 ]
```

## Notes

- Please [report any issues or
  bugs](https://github.com/ropensci/nodbi/issues).
- License: MIT
- Get citation information for `nodbi` in R doing
  `citation(package = 'nodbi')`
- Please note that this package is released with a [Contributor Code of
  Conduct](https://ropensci.org/code-of-conduct/). By contributing to
  this project, you agree to abide by its terms.
- Support for redis has been removed since version 0.5, because no way
  was found to query and update specific documents in a container.
