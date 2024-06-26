# Snow Diff CLI

This is a CLI tool that allows dbt core / dbt Cloud CLI users to quickly see a high-level difference between a table that exists in both the production and development environments. It utilizes the dbt ```project.yml``` file to get the needed Snowflake credentials/schemas and 
then queries from both databases to then numerically compare the outputs of the table. 

## What will it output?

If you are working on existing models and want to compare your changes in the development environment to that in the production environment, you can use this tool to get metrics about the
1. Differences in shape for the two resulting tables
2. Average mean percent difference for each numeric column
3. Average mean frequency ratio for each categorical column

Note:
- Both tables should have the same exact structure and should ideally have correct type casting.

## Default Behaviour:
If columns don't match or data types cannot be adequetely cast to the correct types between tables, snow-diff will output just a comparison of the shapes of the tables. This should theoretically give a solid preview into whether your expectations are in line with the development table vs the production table.


## Requirements:
- Make sure you have the ```project.yml``` file in your ```.dbt``` directry
  - Ex: ```C:\Users\JairusMartinez\.dbt\profiles.yml```
- Include both the development and production dbt environments in the ```project.yml``` file:

```{YAML}
default:
  outputs:
    dev:
      account: your-snowflake-accunt-id
      database: SANDBOX_EDW
      password: your-snowflake-password
      role: ANALYTICS_ENGINEER
      schema: DBT_DEV
      threads: 4
      type: snowflake
      user: your-curaleaf-email
      warehouse: your-snowflake-warehouse
    prod:
      account: your-snowflake-accunt-id
      database: PRD_EDW_DBT
      password: your-snowflake-password
      role: ANALYTICS_ENGINEER
      schema: DBT
      threads: 4
      type: snowflake
      user: your-curaleaf-email
      warehouse: your-snowflake-warehouse
  target: dev
```
- Have dbt-core / dbt-cloud CLI set up
- Have a dedicated virtual environment for dbt to use PowerUser in VSCode

### Virtual Environment Dependencies
- Make sure your pipenv / venv contains:
```
pandas = "*"
numpy = "*"
snowflake-connector-python = "*"
dbt-snowflake = "*"
pyyaml = "*"
```

# Downloading the Snow Diff CLI Package:
1. Activate your dbt virtual environment
2. Make sure the packages above are installed
3. In your dbt virtual environment run the command:
   
   ```
   pip install -e git+https://github.com/jairus-c/snowdiff-cli.git#egg=snowdiff-cli
   ```
5. Run the command ```pip freeze``` in your virtual environment and look for the output below to verify the installation:
   - Can also run ```snow-diff -h``` to verify installation
  ```
  snow-diff @ file:///C:/Users/JairusMartinez/OneDrive%20-%20Curaleaf/Desktop/snowdiff-cli
  -e git+https://github.com/jairus-c/snowdiff-cli.git@6990bfd9f1842c1ae14b56ac127fb8129a9d7e88#egg=snowdiff_cli
  ```
5. To uninstall from your activated virtual environment, run:
   - ```pip uninstall snowdiff-cli```
6. To update, simply run the command from step 4!
  
# Usage
## Commands
```
snow-diff --help
```
- Returns the possible flags and arguments to pass
  ![Screenshot 2024-04-29 205502](https://github.com/jairus-c/snowdiff-cli/assets/165701889/2a6c1106-cd9f-4e2c-85ac-7dbe1b52c3ec)

## Comparing tables from the default schemas
```
snow-diff --table date_dim --filter 'calendar_year = 1970'
```
```
snow-diff -t date_dim -f 'calendar_year = 1970'
```
![Screenshot 2024-04-29 210228](https://github.com/jairus-c/snowdiff-cli/assets/165701889/bcdc848b-fc63-4824-a2b0-49fb47ae24fd)

## Comparing tables from custom schemas
```
snow-diff --table weekly_sales_by_product_details --custom-schema fpa_reporting --filter 'date(weekenddate) > current_date - 30'
```
```
snow-diff -t weekly_sales_by_product_details -cs fpa_reporting -f 'date(weekenddate) > current_date - 30'
```
![Screenshot 2024-04-29 211003](https://github.com/jairus-c/snowdiff-cli/assets/165701889/04449241-e483-4fac-a950-220617a640bb)

# Alternative Usage
You can also simply run ```snow-diff``` in the CLI and you will be prompted to input the table details:

```
Schema, table, and filter are required. Please provide values.
Enter the name of the Snowflake table to compare: **{ENTER SNOWFLAKE TABLE HERE}**
Enter custom schema of the Snowflake table (leave blank if default): **{ENTER CUSTOM SNOWFLAKE SCHEMA HERE}**
Enter a filter condition for the comparison query: **{ENTER FILTER CONDITION TO LIMIT COMPUTE}**
```

1. Querying from the default schema
![Screenshot 2024-04-29 211225](https://github.com/jairus-c/snowdiff-cli/assets/165701889/9a479e66-5381-479b-aecf-b3d530bf8073)
2. Querying from a custom schema
![Screenshot 2024-04-29 211355](https://github.com/jairus-c/snowdiff-cli/assets/165701889/b6b8b87d-5b73-45a8-983c-6990f4b0e693)

# Quirks to Usage
- This tool may have plenty of bugs to discvoer! ;)
  - I have not gotten to creating high coverage unittests for the Python source code
- AEs at Curaleaf do not have access to the production environment in snowflake in terms of writing data and creating jobs; however, you still need to put the prod credentials in the ```profiles.yml``` file so this tool can actually use your credenitals.
- You NEED to know an appropriate column beforehand to filter before you make the comparison
  - using the ```--filter``` flag in the CLI necesitates the use of closed parenthesis but using the user input does not (reference the examples above)
  - A filter condition is needed to run the comparisons
- If the CLI input does not get arguments for the table or filter, it will automatically redirect you to the user input
- If no custom schema is used, the tool will default to the default schemas set in the ```profiles.yml``` file
  - In our case:
      - Prod = ```DBT```
      - Dev = ```DBT_DEV```
- The comparison table MUST exist in both the production environment and development environment to work
- Custom schema names are equivalent to the names set in dbt
  - Therefore, you do not need to include any of the prefixes you see in snowflake
  - ```dbt_fpa_reporting``` and ```dbt_dev_fpa_reporting``` have a custom schema equal to ```fpa_reporting```
- Cannot compare to UAT environment ... yet!

# How are the metrics calculated? ... and what do they mean?

- Shape comparison
  - Compares the difference in the columns and row counts
- Numeric comparison
  - Gets the percent difference across all values between the two tables and calculates the mean of these differences
  - __The closer the mean of these differences is to 0, the better__
- Categorical comparison
  - Gets the frequency ratio of each unique categorical value in a column of one table and divdes it by the same value of the second table
  - This metric shows how aligned the values / value counts of each categorical column is between the two tables
      - Captures frequency/ordinance and presence of distinct values
  - __The closer the mean frequency ratio is to 1, the better__
