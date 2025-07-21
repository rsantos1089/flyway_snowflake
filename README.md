# Connection between flyway and snowflake

![architecture](images/architecture.png)

To generate the successful connection between flyway and snowflake platform using a private connecction , 
we need to make the follow steps:

### Download the docker image of flyway
```console
docker pull redgate/flyway
```

### Create user, database and warehouse in snowflake
Create user and role to use in the connection, assign the necessary privileges to role created. 

```sql
CREATE USER FLYWAY_USER;
create role FLYWAY;
CREATE OR REPLACE DATABASE DEVOPS;
CREATE OR REPLACE SCHEMA FLYWAY;
CREATE WAREHOUSE WH_DEVOPS WITH WAREHOUSE_SIZE = 'XSMALL' WAREHOUSE_TYPE = 'STANDARD' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
ALTER USER FLYWAY_USER SET DEFAULT_ROLE='FLYWAY';
ALTER USER FLYWAY_USER SET DEFAULT_WAREHOUSE='WH_DEVOPS';
```

### Link docker and snowflake user via ssh
Following the official doc: https://docs.snowflake.com/en/user-guide/key-pair-auth
```console 
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
```

```console
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

Open the file rsa_key.pub and copy the value to set in snowflake user propertie

### Setting snowflake user to connect with docker

```sql
ALTER USER FLYWAY_USER SET JDBC_QUERY_RESULT_FORMAT='JSON';
ALTER USER FLYWAY_USER SET RSA_PUBLIC_KEY = '<rsa_key.pub>';
ALTER USER FLYWAY_USER UNSET password;
DESCRIBE USER FLYWAY_USER;
```

### Execute the docker run

To make the connection we need to send the follow params:
* -v (create volume) to send private key
* -v (create volume) to send public key
* -v (create volume) to send folder path with sql scripts
* url that contains snowflake account
* user that contains ssh propertie
* password in blank
* schema to store table of flyway metadata
* command that could  be **migrate** to execute the migration or **repair** when any scrip failed

```console
docker run --rm \
-v C:/Users/ENRIQUE/PycharmProjects/flyway_snowflake/rsa_key.p8:/root/.ssh/rsa_key.p8 \
-v C:/Users/ENRIQUE/PycharmProjects/flyway_snowflake/rsa_key.pub:/root/.ssh/rsa_key.pub \
-v C:/Users/ENRIQUE/PycharmProjects/flyway_snowflake/flyway/sql/co/:/flyway/sql \
redgate/flyway \
-url='jdbc:snowflake://lzuanyu-gcc25290.snowflakecomputing.com:443?db=DEVOPS&warehouse=WH_DEVOPS&private_key_file=/root/.ssh/rsa_key.p8' \
-user=FLYWAY_USER \
-password= \
-schemas=FLYWAY \
migrate
```
