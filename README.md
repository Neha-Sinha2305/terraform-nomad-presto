<p align="center">
 <h2 align="center">Terraform-nomad-presto</h2>
 <p align="center">Terraform module with example</p>
 <a href="https://github.com/fredrikhgrelland/vagrant-hashistack/releases">
   <img alt="Releases" src="https://img.shields.io/badge/dynamic/json?label=with%20vagrant-hashistack&query=%24.current_version.version&url=https%3A%2F%2Fapp.vagrantup.com%2Fapi%2Fv1%2Fbox%2Ffredrikhgrelland%2Fhashistack"/>
 </a>
</p>

#

## Contents
0. [Prerequisites](#prerequisites)
1. [Compatibility](#compatibility)
2. [Usage](#usage)
   1. [Requirements](#requirements)
      1. [Required software](#required-software)
   2. [Providers](#providers)
3. [Inputs](#inputs)
4. [Outputs](#outputs)
5. [Examples](#examples)
6. [Authors](#authors)
7. [License](#license)

## Prerequisites
Please follow [this section in original template](https://github.com/fredrikhgrelland/vagrant-hashistack-template#install-prerequisites)

## Compatibility
|Software|OSS Version|Enterprise Version|
|:---|:---|:---|
|Terraform|0.13.1 or newer||
|Consul|1.8.3 or newer|1.8.3 or newer|
|Vault|1.5.2.1 or newer|1.5.2.1 or newer|
|Nomad|0.12.3 or newer|0.12.3 or newer|

## Usage

```text
make up
```

Check the example of terraform-nomad-presto documentation [here](./example)

### Requirements

#### Required software
See [template README's prerequisites](template_README.md#install-prerequisites).

Local only.
For verification and debugging:
- [consul](https://releases.hashicorp.com/consul/) binary available on `PATH` on the local machine
- java11
- [presto.jar](https://prestosql.io/docs/current/installation/cli.html) in root is version 340

### Providers
This module uses the [Nomad](https://registry.terraform.io/providers/hashicorp/nomad/latest/docs) provider.

## Inputs
| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| nomad\_provider\_address | Nomad provider address | string | "http://127.0.0.1:4646" | yes |
| nomad\_data\_center | Nomad data centers | list(string) | ["dc1"] | yes |
| nomad\_namespace | [Enterprise] Nomad namespace | string | "default" | yes |
| nomad\_job\_name | Nomad job name | string | "presto" | yes |
| service\_name | Presto service name | string | "presto" | yes |
| port | Presto http port | number | 8080 | yes |
| docker\_image | Presto docker image | string | "prestosql/presto:333" | yes |
| container\_environment\_variables | Presto environment variables | list(string) | [""] | no |
| hivemetastore.service_name | Hive metastore service name | string | "hive-metastore" | yes |
| hivemetastore.port | Hive metastore port | number | 9083 | yes |
| minio.service_name | minio service name | string |  | yes |
| minio.port | minio port | number |  | yes |
| minio.access_key | minio access key | string |  | yes |
| minio.secret_key | minio secret key | string |  | yes |


## Outputs
| Name | Description | Type |
|------|-------------|------|
| presto\_service\_name | Presto service name | string |
| presto\_port | Presto port | number |

## Examples
```hcl
module "presto" {
  depends_on = [
    module.minio,
    module.hive
  ]

  source = "github.com/fredrikhgrelland/terraform-nomad-presto.git?ref=0.0.1"

  nomad_job_name    = "presto"
  nomad_datacenters = ["dc1"]
  nomad_namespace   = "default"

  service_name = "presto"
  port         = 8080
  docker_image = "prestosql/presto:333"

  #hivemetastore
  hivemetastore = {
    service_name = module.hive.service_name
    port         = 9083
  }

  # minio
  minio = {
    service_name = module.minio.minio_service_name
    port         = 9000
    access_key   = module.minio.minio_access_key
    secret_key   = module.minio.minio_secret_key
  }
}
```

For detailed information check [example/](./example) directory.

### Verifying setup

You can verify successful run with next steps:

#### Option 1 [hive-metastore and nomad]

* Go to [http://localhost:4646/ui/exec/hive-metastore](http://localhost:4646/ui/exec/hive-metastore)
* Chose metastoreserver -> metastoreserver and click enter.
* Connect using beeline cli
```text
# from metastore (loopback)
beeline -u jdbc:hive2://
```
* Query existing tables (beeline-cli)

```text
SHOW DATABASES;
SHOW TABLES IN <database-name>;
DROP DATABASE <database-name>;
SELECT * FROM <table_name>;

# examples
SHOW TABLES;
SELECT * FROM iris;
SELECT * FROM tweets;
```

#### Option 2 [presto and nomad]
* Go to [http://localhost:4646/ui/exec/presto](http://localhost:4646/ui/exec/presto)
* Chose standalone -> server and click enter.
* Connect using presto-cli
```text
presto
```
* Query existing tables (presto-cli)
```text
SHOW CATALOGS [ LIKE pattern ]
SHOW SCHEMAS [ FROM catalog ] [ LIKE pattern ]
SHOW TABLES [ FROM schema ] [ LIKE pattern ]

# examples
SHOW CATALOGS;
SHOW SCHEMAS IN hive;
SHOW TABLES IN hive.default;
SELECT * FROM hive.default.iris;
```

#### Option 3 [local presto-cli]
`NB!` Check [required software section](#required-software) first.

* create local proxy to presto instance with `consul` binary.
```text
make proxy-presto
```

* in another terminal run `presto-cli` session
```text
presto --server localhost:8080 --catalog hive --schema default --user presto
```

* Query tables (3 tables should be available)
```text
show tables;
select * from <table>;
```

To debug or continue developing you can use [presto cli](https://prestosql.io/docs/current/installation/cli.html) locally.
Some useful commands.
```text
# manual table creation for different file types
presto --server localhost:8080 --catalog hive --schema default --user presto --file ./example/resources/query/csv_create_table.sql
presto --server localhost:8080 --catalog hive --schema default --user presto --file ./example/resources/query/json_create_table.sql
presto --server localhost:8080 --catalog hive --schema default --user presto --file ./example/resources/query/avro_tweets_create_table.sql
```

## Authors

## License

________

## References
- [Blog post](https://towardsdatascience.com/load-and-query-csv-file-in-s3-with-presto-b0d50bc773c9)
- Presto, so far (release 340), [supports only varchar columns](https://github.com/prestosql/presto/pull/920#issuecomment-517593414).

## File types

### CSV
```sql
CREATE TABLE iris (
  sepal_length varchar,
  sepal_width varchar,
  petal_length varchar,
  petal_width varchar,
  species varchar
)
WITH (
  format = 'CSV',
  external_location='s3a://hive/data/csv/',
  skip_header_line_count=1
);
```

`NB!` Hive supports csv int types for columns.
You can create a table for `csv` file format using `hive-metastore`.
```sql
CREATE EXTERNAL TABLE iris (sepal_length DECIMAL, sepal_width DECIMAL,
petal_length DECIMAL, petal_width DECIMAL, species STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
LOCATION 's3a://hive/data/csv/'
TBLPROPERTIES ("skip.header.line.count"="1");
```

### JSON

```sql
CREATE TABLE somejson (
  description varchar,
  foo ROW (
    bar varchar,
    quux varchar,
    level1 ROW (
      l2string varchar,
      l2struct ROW (
        level3 varchar
      )
    )
  ),
  wibble varchar,
  wobble ARRAY (
    ROW (
      entry int,
      EntryDetails ROW (
        details varchar,
        details2 int
      )
    )
  )
)
WITH (
  format = 'JSON',
  external_location = 's3a://hive/data/json/'
);
```

### AVRO

```sql
CREATE TABLE tweets (
  username varchar,
  tweet varchar,
  timestamp bigint
)
WITH (
  format = 'AVRO',
  external_location='s3a://hive/data/avro-tweet/'
);
```

### PROTOBUF
Reference to [using-protobuf-parquet](https://costimuraru.wordpress.com/2018/04/26/using-protobuf-parquet-with-aws-athena-presto-or-hive/)

todo
```sql

```
