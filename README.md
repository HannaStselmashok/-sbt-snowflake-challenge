# DBT core + Snowflake

## Set up the environment

### Create dbt user, database and schema in Snowflake

```sql
-- Use an admin role
USE ROLE accountadmin;

-- dbt roles
CREATE OR REPLACE ROLE dbt_dev_role;
CREATE OR REPLACE ROLE dbt_prod_role;

-- Create the 'dbt' user and assign to role
CREATE USER IF NOT EXISTS dbt_user
  PASSWORD='dbtPassword123'
  LOGIN_NAME='dbt'
  MUST_CHANGE_PASSWORD=FALSE
  COMMENT='DBT user used for data transformation';

GRANT ROLE dbt_dev_role,dbt_prod_role TO USER dbt_user;
GRANT ROLE dbt_dev_role,dbt_prod_role TO ROLE sysadmin;

-- dbt objects
USE ROLE sysadmin;

CREATE OR REPLACE WAREHOUSE dbt_dev_wh  WITH WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 INITIALLY_SUSPENDED = TRUE;
CREATE OR REPLACE WAREHOUSE dbt_dev_heavy_wh  WITH WAREHOUSE_SIZE = 'LARGE' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 INITIALLY_SUSPENDED = TRUE;
CREATE OR REPLACE WAREHOUSE dbt_prod_wh WITH WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 INITIALLY_SUSPENDED = TRUE;
CREATE OR REPLACE WAREHOUSE dbt_prod_heavy_wh  WITH WAREHOUSE_SIZE = 'LARGE' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 INITIALLY_SUSPENDED = TRUE;

GRANT ALL ON WAREHOUSE dbt_dev_wh  TO ROLE dbt_dev_role;
GRANT ALL ON WAREHOUSE dbt_dev_heavy_wh  TO ROLE dbt_dev_role;
GRANT ALL ON WAREHOUSE dbt_prod_wh TO ROLE dbt_prod_role;
GRANT ALL ON WAREHOUSE dbt_prod_heavy_wh  TO ROLE dbt_prod_role;

CREATE OR REPLACE DATABASE dbt_hol_dev;
CREATE OR REPLACE DATABASE dbt_hol_prod;
CREATE OR REPLACE SCHEMA  dbt_hol_dev;
CREATE OR REPLACE SCHEMA  dbt_hol_prod;

GRANT ALL ON DATABASE dbt_hol_dev  TO ROLE dbt_dev_role;
GRANT ALL ON DATABASE dbt_hol_prod TO ROLE dbt_prod_role;
GRANT ALL ON ALL SCHEMAS IN DATABASE dbt_hol_dev   TO ROLE dbt_dev_role;
GRANT ALL ON ALL SCHEMAS IN DATABASE dbt_hol_prod  TO ROLE dbt_prod_role;
```

As result of these steps, we should have:

- two empty databases: PROD, DEV
- two pair of virtual warehouses: two for prod, two for dev workloads
- a pair of roles and one user

### Set up virtual environment

```shell
pip install virtualenv
virtualenv dbtsnowflake
dbtsnowflake\Scripts\activate
```

### Install dbt
```shell
pip install dbt-snowflake==1.7.2
dbt init dbt_hol
cd dbt_hol
```

- account: lkhpmcc-cn69015
- user: dbt
- password: dbtPassword123
- role: dbt_dev_role
- warehouse: dbt_dev_wh
- database: dbt_hol_dev
- schema: public
- threads: 200

Then to connect run `dbt debug` and `dbt run` to run sample models that comes with dbt templates by default to validate everything is set up correctly.
You can then check these models in Snowflake account.

If errors:
- check credentials in profiles.yml
- check that profile name in dbt_project.yml is the same as in profiles.yml

## Connect to data source

Here is the [data set](https://app.snowflake.com/marketplace/listing/GZTSZAS2KF7/cybersyn-inc-financial-economic-essentials?available=installed) I'm using for this project.

In the pop-up, leave the database name as proposed by default (important!), check the "I accept..." box and then add PUBLIC role to the additional roles.

## Build DBT data pipelines

1. Create separate folders for models, representing logical levels in the pipelins

```shell
mkdir models/l10_staging
mkdir models/l20_transform
mkdir models/l30_mart
mkdir models/tests
```

2. Modify dbt_project.yml file to reflect model structure

```yaml
models:
  dbt_hol:
      # Applies to all files under models/example/
      example:
          materialized: view
          +enabled: false
      l10_staging:
          schema: l10_staging
          materialized: view
      l20_transform:
          schema: l20_transform
          materialized: view
      l30_mart:
          schema: l30_mart
          materialized: view
```

3. Custom schema name. By default dbt generated a schema name by appending it to the target schema env name (dev, prod). To make the names between dev and prod databases the same, we need to create sql file in macros schema_name.sql and place this jinja script there.

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}


{% macro set_query_tag() -%}
  {% set new_query_tag = model.name %} {# always use model name #}
  {% if new_query_tag %}
    {% set original_query_tag = get_current_query_tag() %}
    {{ log("Setting query_tag to '" ~ new_query_tag ~ "'. Will reset to '" ~ original_query_tag ~ "' after materialization.") }}
    {% do run_query("alter session set query_tag = '{}'".format(new_query_tag)) %}
    {{ return(original_query_tag)}}
  {% endif %}
  {{ return(none)}}
{% endmacro %}
```

There is another macro overridden in the file: set_query_tag(). This one provides the ability to add additional level of transparency by automatically setting Snowflake query_tag to the name of the model it associated with.

So if you go in Snowflake UI and click ‘History' icon on top, you are going to see all SQL queries run on Snowflake account(successful, failed, running etc) and clearly see what dbt model this particular query is related to.

4. DBT plugins. Let's install dbt package dbt_utils. For that create packages.yml file in the root of dbt project folder.

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

Run `dbt deps` to install the package.
