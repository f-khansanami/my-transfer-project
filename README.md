# README

DBT Project - Enhancements
Step 1: Update dbt_project.yml
Instead of passing environment-specific settings through a loop, define them directly in dbt_project.yml using vars. Here’s how to set it up:

In dbt_project.yml add the vars section.
Structure the vars section to define environment-specific configurations for each model based on the target.
Inside the models, we can access these settings based on the target, using {{ var('model_config')[target.name]['some_setting'] }}
Step 2: Configure table_config.yml and preprod_table_config.yml Variables in dbt_project.yml
Since there are many settings in table_config.yml and preprod_table_config.yml, we can add them as variables under the respective environments in the vars section.

Copy the configurations from table_config.yml and preprod_table_config.yml and place them under vars in dbt_project.yml with specific keys.
In our models, access the environment configurations like this: {{ var('dev_config')['setting1'] if target.name == 'dev' else var('prod_config')['setting1'] }}
dbt_project.yml
name: 'dap'
version: '1.0.0'
config-version: 2

# This setting configures which "profile" dbt uses for this project.
profile: 'domain_dbt'

# These configurations specify where dbt should look for different types of files.
# The `source-paths` config, for example, states that models in this project can be
# found in the "models/" directory. You probably won't need to change these!
source-paths: ["models"]
analysis-paths: ["analysis"]
#test-paths: ["tests"]
data-paths: ["data"]
macro-paths: ["../../dags/domain_utils/macros"]
snapshot-paths: ["snapshots"]

target-path: "target"  # directory which will store compiled SQL files
clean-targets:         # directories to be removed by `dbt clean`
    - "target"
    - "dbt_modules"

models:
    dap:
        enabled: true
        materialized: table
    my_transfer_project_refactor:
        reports:
            +enabled: "{{ target.name == 'preprod' }}"

# Define environment-specific variables
vars:
    # preprod configuration
    preprod_config:
        m_reports:
            DBT_SRC_GCP_PROJECT: "project id"
            DBT_SRC_GCP_DATASET: "coreruby"
            DBT_GCP_DATASET: "transfer_project_preprod"

        default:
            DBT_SRC_GCP_PROJECT: "project id"
            DBT_SRC_GCP_DATASET: "coreruby"
            DBT_GCP_DATASET: "transfer_project_preprod"
            DBT_SRC_TABLE_NAME: "coreruby_transactions_native"
            DBT_TGT_TABLE_NAME: "coreruby_transactions_native_raw"

    # Production configuration
    prod_config:
        m_reports:
            DBT_SRC_GCP_PROJECT: "project id"
            DBT_SRC_GCP_DATASET: "coreruby"
            DBT_GCP_DATASET: "transfer_project_preprod"

        default:
            DBT_SRC_GCP_PROJECT: "project id"
            DBT_SRC_GCP_DATASET: "coreruby"
            DBT_GCP_DATASET: "transfer_project_preprod"
            DBT_SRC_TABLE_NAME: "coreruby_transactions_native"
            DBT_TGT_TABLE_NAME: "coreruby_transactions_native_raw_prod"
   
Step 3: Add Conditional Enabling of Models
Use DBT’s enabled configuration to conditionally enable models based on the target environment:
Open each model file that needs conditional execution.

Add a configuration block at the top of each file to enable or disable or pass env var based on the env target

models/reports/default.sql: In this model, the vars are picked up based on the condition defined in Jinja template. Appropriate env vars are fetched when run via dbt run --target prod or dbt run --target preprod

{{ config(
    alias = var("preprod_config")["default"]["DBT_TGT_TABLE_NAME"] if target.name == "preprod" else var("prod_config")["default"]["DBT_TGT_TABLE_NAME"]
) }}

SELECT *  
models/reports/m0_report.sql: This model will only run if dbt run --target preprod

{{ config(
    enabled = target.name == 'preprod'
) }}

with coreruby_transactions_unique as (
select t.* from (
select


models/reports/m1_report.sql: This model will only run if dbt run --target prod

{{ config(
    enabled = target.name == 'prod'
) }}

with coreruby_transaction_details as (
select t.* from
(
select *except(name),
upper(name) as name,

