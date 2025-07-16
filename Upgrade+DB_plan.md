# Major DB Upgrade 

## Commands
```
# get CAS namespaces
kubectl get namespaces | grep hmpps-community-accomodation

## get upgrade versions 
aws rds describe-db-engine-versions --engine postgres  --engine-version 14 --query "DBEngineVersions[*].ValidUpgradeTarget[*].{EngineVersion:EngineVersion}" --output text

# what is the default 
aws rds describe-db-engine-versions --default-only --engine postgres    
```

## URLS
db instance class support 
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.Support.html

## Environments

db.t3.small = Supports 16, 17
db.t3.xlarge = Supports 16, 17

### DEV
namespace :  hmpps-community-accomodation-dev

```
db_engine_version            = "14"
db_instance_class            = "db.t3.small"
rds_family                   = "postgres14"
allow_major_version_upgrade  = "true"
```

### DEMO

namespace :  hmpps-community-accomodation-demo

```
  db_engine_version            = "14"
  db_instance_class            = "db.t3.small"
  rds_family                   = "postgres14"
  allow_major_version_upgrade  = "true"
```

### TEST

namespace :  hmpps-community-accomodation-test

```
  db_engine_version            = "14"
  db_instance_class            = "db.t3.small"
  rds_family                   = "postgres14"
  allow_major_version_upgrade  = "true"
```

### PRE PROD

namespace :  hmpps-community-accomodation-preprod

```
  db_engine_version            = "14"
  db_instance_class            = "db.t3.xlarge"
  rds_family                   = "postgres14"
  allow_major_version_upgrade  = "false"
```

### PROD

namespace :  hmpps-community-accomodation-prod

```
  db_engine_version            = "14"
  db_instance_class            = "db.t3.xlarge"
  rds_family                   = "postgres14"
  allow_major_version_upgrade  = "false"
```


### For Each namespace 

1. clone https://github.com/ministryofjustice/cloud-platform-environments, 

2. create a new branch

    1. Create an empty file (namespace will be ignored)
    ```
    touch namespaces/live.cloud-platform.service.justice.gov.uk/{mynamespace}/APPLY_PIPELINE_SKIP_THIS_NAMESPACE
    ```

    2. Create a PR / Merge

3. Create a new branch 

    1. prepare_for_major_upgrade = true  // what is this? is this allow_major_version_upgrade
        db_engine                 = "postgres"
        db_engine_version         = "16.1"
        rds_family                = "postgres16"
    2. create PR / Merge

4. Createa new branch 
     
     prepare_for_major_upgrade = false
    rm namespaces/live.cloud-platform.service.justice.gov.uk/{mynamespace}/APPLY_PIPELINE_SKIP_THIS_NAMESPACE
