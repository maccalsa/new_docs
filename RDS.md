# Cloud Platform RDS Instance Module

[![Releases](https://img.shields.io/github/v/release/ministryofjustice/cloud-platform-terraform-rds-instance.svg)](https://github.com/ministryofjustice/cloud-platform-terraform-rds-instance/releases)

## 🎯 What is this project?

This is a **Terraform module** that creates and manages **Amazon RDS (Relational Database Service) instances** for the UK Ministry of Justice's Cloud Platform. Think of it as a "database factory" that automatically sets up secure, compliant database servers in the cloud.

### 🤔 What does that mean in simple terms?

- **Terraform**: A tool that lets you describe your cloud infrastructure as code (like a recipe for building servers)
- **Module**: A reusable package of Terraform code that creates specific resources
- **RDS**: Amazon's managed database service (you don't have to install or maintain the database software yourself)
- **Cloud Platform**: The Ministry of Justice's internal cloud environment

## 🏗️ What does this module create?

When you use this module, it automatically creates:

### 🗄️ **The Database Server**
- A fully managed database instance (PostgreSQL, MySQL, SQL Server, etc.)
- Automatic backups and security updates
- Encryption for all your data
- High availability (can survive hardware failures)

### 🔒 **Security Infrastructure**
- **Encryption keys** to protect your data
- **Security groups** (like firewalls) to control who can access the database
- **Network isolation** in private subnets
- **SSL/TLS encryption** for all connections

### 🛠️ **Management Tools**
- **IAM policies** for secure programmatic access
- **Parameter groups** for database configuration
- **Subnet groups** for network placement
- **Snapshot capabilities** for backups and migrations

## 🎨 What database types are supported?

This module supports multiple database engines:

| Database Type | Use Case | Example Version |
|---------------|----------|-----------------|
| **PostgreSQL** | Modern web applications, complex queries | 15.x |
| **MySQL** | Web applications, content management | 8.0 |
| **MariaDB** | MySQL alternative, open source | 10.x |
| **SQL Server** | Windows applications, enterprise systems | 2019 |
| **Oracle** | Enterprise applications, legacy systems | SE2 |

## 🔧 How to use this module

### Basic Setup

Here's the simplest way to create a database:

```hcl
module "my_database" {
  source = "github.com/ministryofjustice/cloud-platform-terraform-rds-instance?ref=version"

  # Where to put the database
  vpc_name = "live-1"

  # What type of database
  db_engine         = "postgres"
  db_engine_version = "15"
  rds_family        = "postgres15"
  db_instance_class = "db.t4g.micro"

  # Who owns this (required for billing and support)
  business_unit          = "digital-prisons"
  application            = "my-app"
  is_production          = "false"
  team_name              = "my-team"
  namespace              = "my-namespace"
  environment_name       = "development"
  infrastructure_support = "my-team@digital.justice.gov.uk"
}
```

### What each setting means:

- **`vpc_name`**: Which network to put the database in
- **`db_engine`**: What type of database (postgres, mysql, etc.)
- **`db_engine_version`**: Which version of the database software
- **`db_instance_class`**: How powerful the database server should be
- **`business_unit`**: Which part of the Ministry of Justice this is for
- **`application`**: What application will use this database
- **`is_production`**: Whether this is for live users or testing
- **`team_name`**: Which team is responsible
- **`namespace`**: The Kubernetes namespace (like a folder for your app)
- **`environment_name`**: What environment (dev, staging, prod)
- **`infrastructure_support`**: Who to contact for support

## 🔐 How to access your database

### Getting Credentials

The module automatically creates a **Kubernetes secret** with your database credentials. You can get them like this:

```bash
kubectl -n [your namespace] get secret [secret name] -o yaml | ./decode.rb
```

This will show you:
- Database hostname (where to connect)
- Username and password
- Port number
- Database name

### Connecting from your application

Your application should always get credentials from the Kubernetes secret, not hardcode them. The secret looks like this:

```yaml
apiVersion: v1
data:
  database_name: your-database-name
  database_password: your-password
  database_username: your-username
  rds_instance_address: cloud-platform-xxxxx.yyyyy.eu-west-2.rds.amazonaws.com
  rds_instance_endpoint: cloud-platform-xxxxx.yyyyy.eu-west-2.rds.amazonaws.com:5432
  rds_instance_port: '5432'
```

### Connecting manually (for debugging)

You can connect to your database using a tool like `psql`:

```bash
# Start a database client in the cluster
kubectl -n [your namespace] run shell --rm -i --tty --image bitnami/postgresql -- bash

# Connect to your database
psql -h [rds_instance_address] -U [database_username] [database_name]
```

## 💰 Cost-saving features

### Auto Start/Stop (for non-production)

You can enable automatic shutdown during off-hours to save money:

```hcl
module "my_database" {
  # ... other settings ...
  enable_rds_auto_start_stop = true  # Shuts down 10 PM - 6 AM
}
```

### Storage Autoscaling

The database can automatically grow storage as needed:

```hcl
module "my_database" {
  # ... other settings ...
  db_allocated_storage     = 20    # Start with 20 GB
  db_max_allocated_storage = 500   # Can grow up to 500 GB
}
```

## 🔄 Advanced features

### Creating from a backup

If you have an existing database backup, you can restore from it:

```hcl
module "my_database" {
  # ... other settings ...
  snapshot_identifier = "arn:aws:rds:region:account:snapshot:snapshot-name"
}
```

### Creating a read replica

You can create a copy of your database for read-only operations:

```hcl
module "my_database_replica" {
  # ... other settings ...
  replicate_source_db = "original-database-identifier"
}
```

### Custom database settings

You can configure specific database parameters:

```hcl
module "my_database" {
  # ... other settings ...
  db_parameter = [
    {
      name         = "max_connections"
      value        = "200"
      apply_method = "pending-reboot"
    }
  ]
}
```

## 🛡️ Security features

### What's automatically secured:

- **Encryption at rest**: All data is encrypted when stored
- **Encryption in transit**: All connections use SSL/TLS
- **Network isolation**: Database is in private subnets
- **Access control**: Only authorized applications can connect
- **IAM integration**: Secure programmatic access

### Additional security options:

```hcl
module "my_database" {
  # ... other settings ...
  deletion_protection = true  # Prevents accidental deletion
  performance_insights_enabled = true  # Monitor performance
}
```

## 📊 Monitoring and management

### Performance Insights

Enable detailed performance monitoring:

```hcl
module "my_database" {
  # ... other settings ...
  performance_insights_enabled = true
}
```

### Automated backups

Configure backup retention:

```hcl
module "my_database" {
  # ... other settings ...
  db_backup_retention_period = 30  # Keep backups for 30 days
  backup_window = "03:00-04:00"    # Backup at 3 AM UTC
}
```

### Maintenance windows

Schedule maintenance during low-traffic periods:

```hcl
module "my_database" {
  # ... other settings ...
  maintenance_window = "Sun:02:00-Sun:04:00"  # Sunday 2-4 AM UTC
}
```

## 🚨 Important notes

### Always use Kubernetes secrets

**Never** hardcode database credentials in your application. Always get them from the Kubernetes secret created by this module. The secret is automatically updated when credentials change.

### Production considerations

For production databases:
- Set `is_production = "true"`
- Enable `deletion_protection = true`
- Use appropriate instance sizes
- Consider enabling `performance_insights_enabled = true`

### Storage requirements

Different storage types have minimum requirements:
- **gp3**: Minimum 20 GB
- **io2**: Minimum 100 GB (20 GB for SQL Server)
- **io2 with high IOPS**: Requires specific IOPS settings

## 🔗 Related resources

- [Cloud Platform User Guide](https://user-guide.cloud-platform.service.justice.gov.uk/)
- [Amazon RDS Documentation](https://docs.aws.amazon.com/rds/)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

## 📝 Examples

See the `examples/` folder for complete working examples:
- `rds-postgresql.tf` - PostgreSQL database setup
- `rds-mysql.tf` - MySQL database setup
- `rds-mssql.tf` - SQL Server database setup
- `rds-mariadb.tf` - MariaDB database setup

## 🤝 Getting help

If you need help with this module:
1. Check the [Cloud Platform User Guide](https://user-guide.cloud-platform.service.justice.gov.uk/)
2. Look at the examples in the `examples/` folder
3. Contact your infrastructure support team
4. Check the [technical documentation](#technical-documentation) below

---

## Technical Documentation

*The following sections contain detailed technical information for advanced users.*

### Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.2.5 |
| random | >= 2.0.0 |

### Providers

| Name | Version |
|------|---------|
| aws | n/a |
| random | >= 2.0.0 |

### Resources

| Name | Type |
|------|------|
| [aws_db_instance.rds](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) | resource |
| [aws_db_parameter_group.custom_parameters](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_parameter_group) | resource |
| [aws_db_snapshot_copy.rds_migration_snapshot](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_snapshot_copy) | resource |
| [aws_db_subnet_group.db_subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group) | resource |
| [aws_iam_policy.irsa](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_kms_alias.alias](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_alias) | resource |
| [aws_kms_key.kms](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) | resource |
| [aws_security_group.rds-sg](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [random_id.id](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/id) | resource |
| [random_password.password](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password) | resource |
| [random_string.username](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) | resource |

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| allow_major_version_upgrade | Indicates that major version upgrades are allowed. | `string` | `"false"` | no |
| allow_minor_version_upgrade | Indicates that minor version upgrades are allowed. | `string` | `"true"` | no |
| application | Application name | `string` | n/a | yes |
| backup_window | The daily time range (in UTC) during which automated backups are created if they are enabled. Example: 09:46-10:16 | `string` | `""` | no |
| business_unit | Area of the MOJ responsible for the service | `string` | n/a | yes |
| ca_cert_identifier | Specifies the identifier of the CA certificate for the DB instance | `string` | `"rds-ca-rsa2048-g1"` | no |
| character_set_name | DB char set, used only by MS-SQL | `string` | `"SQL_Latin1_General_CP1_CI_AS"` | no |
| db_allocated_storage | The allocated storage in gibibytes | `number` | `"20"` | no |
| db_backup_retention_period | The days to retain backups. Must be 1 or greater to be a source for a Read Replica | `string` | `"7"` | no |
| db_engine | Database engine used e.g. postgres, mysql, sqlserver-ex | `string` | `"postgres"` | no |
| db_engine_version | The engine version to use e.g. 13.2 for Postgresql, 8.0 for MySQL, 15.00.4073.23.v1 for MS-SQL. Omitting the minor release part allows for automatic updates. | `string` | `"10"` | no |
| db_instance_class | The instance type of the RDS instance | `string` | `"db.t2.small"` | no |
| db_iops | The amount of provisioned IOPS. | `number` | `null` | no |
| db_max_allocated_storage | Maximum storage limit for storage autoscaling | `string` | `"10000"` | no |
| db_name | The name of the database to be created on the instance (if empty, it will be the generated random identifier) | `string` | `""` | no |
| db_parameter | A list of DB parameters to apply. Note that parameters may differ from a DB family to another | `list(object({apply_method = string, name = string, value = string}))` | `[{apply_method = "immediate", name = "rds.force_ssl", value = "1"}]` | no |
| db_password_rotated_date | Using this variable will spin new db password by providing date as value | `string` | `""` | no |
| deletion_protection | (Optional) If the DB instance should have deletion protection enabled. The database can't be deleted when this value is set to true. The default is false. | `string` | `"false"` | no |
| enable_rds_auto_start_stop | Enable auto start and stop of the RDS instances during 10:00 PM - 6:00 AM for cost saving | `bool` | `false` | no |
| environment_name | Environment name | `string` | n/a | yes |
| infrastructure_support | The team responsible for managing the infrastructure. Should be of the form <team-name> (<team-email>) | `string` | n/a | yes |
| is_migration | Set this to true if you are creating a new RDS instance using an existing snapshot (via snapshot_identifier) | `bool` | `false` | no |
| is_production | Whether this is used for production or not | `string` | n/a | yes |
| license_model | License model information for this DB instance, options for MS-SQL are: license-included \| bring-your-own-license \| general-public-license | `string` | `null` | no |
| maintenance_window | The window to perform maintenance in. Syntax: 'ddd:hh24:mi-ddd:hh24:mi'. Eg: 'Mon:00:00-Mon:03:00' | `string` | `null` | no |
| namespace | Namespace name | `string` | n/a | yes |
| option_group_name | (Optional) The name of an 'aws_db_option_group' to associate to the DB instance | `string` | `null` | no |
| performance_insights_enabled | Enable performance insights for RDS? Note: the user should ensure insights are disabled once the desired outcome is achieved. | `bool` | `false` | no |
| prepare_for_major_upgrade | Set this to true to change your parameter group to the default version, and to turn on the ability to upgrade major versions | `bool` | `false` | no |
| rds_family | Maps the engine version with the parameter group family, a family often covers several versions | `string` | `"postgres10"` | no |
| rds_name | Optional name of the RDS cluster. Changing the name will re-create the RDS | `string` | `""` | no |
| replicate_source_db | Specifies that this resource is a Replicate database, and to use this value as the source database. This correlates to the identifier of another Amazon RDS Database to replicate. | `string` | `null` | no |
| skip_final_snapshot | if false(default), a DB snapshot is created before the DB instance is deleted, using the value from final_snapshot_identifier. If true no DBSnapshot is created | `string` | `"false"` | no |
| snapshot_identifier | Specifies whether or not to create this database from a snapshot. This correlates to the snapshot ID you'd find in the RDS console. | `string` | `""` | no |
| storage_type | One of 'standard' (magnetic), 'gp2' (general purpose SSD), 'gp3' (new generation of general purpose SSD), 'io1' (provisioned IOPS SSD), or 'io2' (new generation of provisioned IOPS SSD). If you specify 'io2', you must also include a value for the 'iops' parameter and the `allocated_storage` must be at least 100 GiB (except for SQL Server which the minimum is 20 GiB). | `string` | `"gp3"` | no |
| team_name | Team name | `string` | n/a | yes |
| vpc_name | The name of the vpc (eg.: cloud-platform-live-0) | `string` | n/a | yes |
| vpc_security_group_ids | (Optional) A list of additional VPC security group IDs to associate with the DB instance - in adition to the default VPC security groups granting access from the Cloud Platform | `list(string)` | `[]` | no |

### Outputs

| Name | Description |
|------|-------------|
| database_name | Name of the database |
| database_password | Database Password |
| database_username | Database Username |
| db_identifier | The RDS DB Indentifer |
| irsa_policy_arn | IAM policy ARN for access to create database snapshots |
| rds_instance_address | The hostname of the RDS instance |
| rds_instance_endpoint | The connection endpoint in address:port format |
| rds_instance_port | The database port |
| resource_id | RDS Resource ID - used for performance insights (metrics) |

## Tags

Some of the inputs for this module are tags. All infrastructure resources must be tagged to meet the MOJ Technical Guidance on [Documenting owners of infrastructure](https://technical-guidance.service.justice.gov.uk/documentation/standards/documenting-infrastructure-owners.html).

You should use your namespace variables to populate these. See the [Usage](#how-to-use-this-module) section for more information.

## Reading Material

- [Cloud Platform user guide](https://user-guide.cloud-platform.service.justice.gov.uk/#cloud-platform-user-guide)
- [Amazon RDS for MySQL user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)
- [Amazon RDS for PostgreSQL user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [Amazon RDS for MariaDB user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/C







# Cloud Platform RDS Instance Module

[![Releases](https://img.shields.io/github/v/release/ministryofjustice/cloud-platform-terraform-rds-instance.svg)](https://github.com/ministryofjustice/cloud-platform-terraform-rds-instance/releases)

## 🎯 What is this project?

This is a **Terraform module** that creates and manages **Amazon RDS (Relational Database Service) instances** for the UK Ministry of Justice's Cloud Platform. Think of it as a "database factory" that automatically sets up secure, compliant database servers in the cloud.

### 🤔 What does that mean in simple terms?

- **Terraform**: A tool that lets you describe your cloud infrastructure as code (like a recipe for building servers)
- **Module**: A reusable package of Terraform code that creates specific resources
- **RDS**: Amazon's managed database service (you don't have to install or maintain the database software yourself)
- **Cloud Platform**: The Ministry of Justice's internal cloud environment

## 🏗️ What does this module create?

When you use this module, it automatically creates:

### 🗄️ **The Database Server**
- A fully managed database instance (PostgreSQL, MySQL, SQL Server, etc.)
- Automatic backups and security updates
- Encryption for all your data
- High availability (can survive hardware failures)

### 🔒 **Security Infrastructure**
- **Encryption keys** to protect your data
- **Security groups** (like firewalls) to control who can access the database
- **Network isolation** in private subnets
- **SSL/TLS encryption** for all connections

### 🛠️ **Management Tools**
- **IAM policies** for secure programmatic access
- **Parameter groups** for database configuration
- **Subnet groups** for network placement
- **Snapshot capabilities** for backups and migrations

## 🎨 What database types are supported?

This module supports multiple database engines:

| Database Type | Use Case | Example Version |
|---------------|----------|-----------------|
| **PostgreSQL** | Modern web applications, complex queries | 15.x |
| **MySQL** | Web applications, content management | 8.0 |
| **MariaDB** | MySQL alternative, open source | 10.x |
| **SQL Server** | Windows applications, enterprise systems | 2019 |
| **Oracle** | Enterprise applications, legacy systems | SE2 |

## 🔧 How to use this module

### Basic Setup

Here's the simplest way to create a database:

```hcl
module "my_database" {
  source = "github.com/ministryofjustice/cloud-platform-terraform-rds-instance?ref=version"

  # Where to put the database
  vpc_name = "live-1"

  # What type of database
  db_engine         = "postgres"
  db_engine_version = "15"
  rds_family        = "postgres15"
  db_instance_class = "db.t4g.micro"

  # Who owns this (required for billing and support)
  business_unit          = "digital-prisons"
  application            = "my-app"
  is_production          = "false"
  team_name              = "my-team"
  namespace              = "my-namespace"
  environment_name       = "development"
  infrastructure_support = "my-team@digital.justice.gov.uk"
}
```

### What each setting means:

- **`vpc_name`**: Which network to put the database in
- **`db_engine`**: What type of database (postgres, mysql, etc.)
- **`db_engine_version`**: Which version of the database software
- **`db_instance_class`**: How powerful the database server should be
- **`business_unit`**: Which part of the Ministry of Justice this is for
- **`application`**: What application will use this database
- **`is_production`**: Whether this is for live users or testing
- **`team_name`**: Which team is responsible
- **`namespace`**: The Kubernetes namespace (like a folder for your app)
- **`environment_name`**: What environment (dev, staging, prod)
- **`infrastructure_support`**: Who to contact for support

## �� How to access your database

### Getting Credentials

The module automatically creates a **Kubernetes secret** with your database credentials. You can get them like this:

```bash
kubectl -n [your namespace] get secret [secret name] -o yaml | ./decode.rb
```

This will show you:
- Database hostname (where to connect)
- Username and password
- Port number
- Database name

### Connecting from your application

Your application should always get credentials from the Kubernetes secret, not hardcode them. The secret looks like this:

```yaml
apiVersion: v1
data:
  database_name: your-database-name
  database_password: your-password
  database_username: your-username
  rds_instance_address: cloud-platform-xxxxx.yyyyy.eu-west-2.rds.amazonaws.com
  rds_instance_endpoint: cloud-platform-xxxxx.yyyyy.eu-west-2.rds.amazonaws.com:5432
  rds_instance_port: '5432'
```

### Connecting manually (for debugging)

You can connect to your database using a tool like `psql`:

```bash
# Start a database client in the cluster
kubectl -n [your namespace] run shell --rm -i --tty --image bitnami/postgresql -- bash

# Connect to your database
psql -h [rds_instance_address] -U [database_username] [database_name]
```

## 💰 Cost-saving features

### Auto Start/Stop (for non-production)

You can enable automatic shutdown during off-hours to save money:

```hcl
module "my_database" {
  # ... other settings ...
  enable_rds_auto_start_stop = true  # Shuts down 10 PM - 6 AM
}
```

### Storage Autoscaling

The database can automatically grow storage as needed:

```hcl
module "my_database" {
  # ... other settings ...
  db_allocated_storage     = 20    # Start with 20 GB
  db_max_allocated_storage = 500   # Can grow up to 500 GB
}
```

## 🔄 Advanced features

### Creating from a backup

If you have an existing database backup, you can restore from it:

```hcl
module "my_database" {
  # ... other settings ...
  snapshot_identifier = "arn:aws:rds:region:account:snapshot:snapshot-name"
}
```

### Creating a read replica

You can create a copy of your database for read-only operations:

```hcl
module "my_database_replica" {
  # ... other settings ...
  replicate_source_db = "original-database-identifier"
}
```

### Custom database settings

You can configure specific database parameters:

```hcl
module "my_database" {
  # ... other settings ...
  db_parameter = [
    {
      name         = "max_connections"
      value        = "200"
      apply_method = "pending-reboot"
    }
  ]
}
```

## ��️ Security features

### What's automatically secured:

- **Encryption at rest**: All data is encrypted when stored
- **Encryption in transit**: All connections use SSL/TLS
- **Network isolation**: Database is in private subnets
- **Access control**: Only authorized applications can connect
- **IAM integration**: Secure programmatic access

### Additional security options:

```hcl
module "my_database" {
  # ... other settings ...
  deletion_protection = true  # Prevents accidental deletion
  performance_insights_enabled = true  # Monitor performance
}
```

## 📊 Monitoring and management

### Performance Insights

Enable detailed performance monitoring:

```hcl
module "my_database" {
  # ... other settings ...
  performance_insights_enabled = true
}
```

### Automated backups

Configure backup retention:

```hcl
module "my_database" {
  # ... other settings ...
  db_backup_retention_period = 30  # Keep backups for 30 days
  backup_window = "03:00-04:00"    # Backup at 3 AM UTC
}
```

### Maintenance windows

Schedule maintenance during low-traffic periods:

```hcl
module "my_database" {
  # ... other settings ...
  maintenance_window = "Sun:02:00-Sun:04:00"  # Sunday 2-4 AM UTC
}
```

## 🚨 Important notes

### Always use Kubernetes secrets

**Never** hardcode database credentials in your application. Always get them from the Kubernetes secret created by this module. The secret is automatically updated when credentials change.

### Production considerations

For production databases:
- Set `is_production = "true"`
- Enable `deletion_protection = true`
- Use appropriate instance sizes
- Consider enabling `performance_insights_enabled = true`

### Storage requirements

Different storage types have minimum requirements:
- **gp3**: Minimum 20 GB
- **io2**: Minimum 100 GB (20 GB for SQL Server)
- **io2 with high IOPS**: Requires specific IOPS settings

## 🔗 Related resources

- [Cloud Platform User Guide](https://user-guide.cloud-platform.service.justice.gov.uk/)
- [Amazon RDS Documentation](https://docs.aws.amazon.com/rds/)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

## 📝 Examples

See the `examples/` folder for complete working examples:
- `rds-postgresql.tf` - PostgreSQL database setup
- `rds-mysql.tf` - MySQL database setup
- `rds-mssql.tf` - SQL Server database setup
- `rds-mariadb.tf` - MariaDB database setup

## 🤝 Getting help

If you need help with this module:
1. Check the [Cloud Platform User Guide](https://user-guide.cloud-platform.service.justice.gov.uk/)
2. Look at the examples in the `examples/` folder
3. Contact your infrastructure support team
4. Check the [technical documentation](#technical-documentation) below

---

## Technical Documentation

*The following sections contain detailed technical information for advanced users.*

### Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.2.5 |
| random | >= 2.0.0 |

### Providers

| Name | Version |
|------|---------|
| aws | n/a |
| random | >= 2.0.0 |

### Resources

| Name | Type |
|------|------|
| [aws_db_instance.rds](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) | resource |
| [aws_db_parameter_group.custom_parameters](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_parameter_group) | resource |
| [aws_db_snapshot_copy.rds_migration_snapshot](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_snapshot_copy) | resource |
| [aws_db_subnet_group.db_subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group) | resource |
| [aws_iam_policy.irsa](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_kms_alias.alias](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_alias) | resource |
| [aws_kms_key.kms](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) | resource |
| [aws_security_group.rds-sg](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [random_id.id](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/id) | resource |
| [random_password.password](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password) | resource |
| [random_string.username](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) | resource |

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| allow_major_version_upgrade | Indicates that major version upgrades are allowed. | `string` | `"false"` | no |
| allow_minor_version_upgrade | Indicates that minor version upgrades are allowed. | `string` | `"true"` | no |
| application | Application name | `string` | n/a | yes |
| backup_window | The daily time range (in UTC) during which automated backups are created if they are enabled. Example: 09:46-10:16 | `string` | `""` | no |
| business_unit | Area of the MOJ responsible for the service | `string` | n/a | yes |
| ca_cert_identifier | Specifies the identifier of the CA certificate for the DB instance | `string` | `"rds-ca-rsa2048-g1"` | no |
| character_set_name | DB char set, used only by MS-SQL | `string` | `"SQL_Latin1_General_CP1_CI_AS"` | no |
| db_allocated_storage | The allocated storage in gibibytes | `number` | `"20"` | no |
| db_backup_retention_period | The days to retain backups. Must be 1 or greater to be a source for a Read Replica | `string` | `"7"` | no |
| db_engine | Database engine used e.g. postgres, mysql, sqlserver-ex | `string` | `"postgres"` | no |
| db_engine_version | The engine version to use e.g. 13.2 for Postgresql, 8.0 for MySQL, 15.00.4073.23.v1 for MS-SQL. Omitting the minor release part allows for automatic updates. | `string` | `"10"` | no |
| db_instance_class | The instance type of the RDS instance | `string` | `"db.t2.small"` | no |
| db_iops | The amount of provisioned IOPS. | `number` | `null` | no |
| db_max_allocated_storage | Maximum storage limit for storage autoscaling | `string` | `"10000"` | no |
| db_name | The name of the database to be created on the instance (if empty, it will be the generated random identifier) | `string` | `""` | no |
| db_parameter | A list of DB parameters to apply. Note that parameters may differ from a DB family to another | `list(object({apply_method = string, name = string, value = string}))` | `[{apply_method = "immediate", name = "rds.force_ssl", value = "1"}]` | no |
| db_password_rotated_date | Using this variable will spin new db password by providing date as value | `string` | `""` | no |
| deletion_protection | (Optional) If the DB instance should have deletion protection enabled. The database can't be deleted when this value is set to true. The default is false. | `string` | `"false"` | no |
| enable_rds_auto_start_stop | Enable auto start and stop of the RDS instances during 10:00 PM - 6:00 AM for cost saving | `bool` | `false` | no |
| environment_name | Environment name | `string` | n/a | yes |
| infrastructure_support | The team responsible for managing the infrastructure. Should be of the form <team-name> (<team-email>) | `string` | n/a | yes |
| is_migration | Set this to true if you are creating a new RDS instance using an existing snapshot (via snapshot_identifier) | `bool` | `false` | no |
| is_production | Whether this is used for production or not | `string` | n/a | yes |
| license_model | License model information for this DB instance, options for MS-SQL are: license-included \| bring-your-own-license \| general-public-license | `string` | `null` | no |
| maintenance_window | The window to perform maintenance in. Syntax: 'ddd:hh24:mi-ddd:hh24:mi'. Eg: 'Mon:00:00-Mon:03:00' | `string` | `null` | no |
| namespace | Namespace name | `string` | n/a | yes |
| option_group_name | (Optional) The name of an 'aws_db_option_group' to associate to the DB instance | `string` | `null` | no |
| performance_insights_enabled | Enable performance insights for RDS? Note: the user should ensure insights are disabled once the desired outcome is achieved. | `bool` | `false` | no |
| prepare_for_major_upgrade | Set this to true to change your parameter group to the default version, and to turn on the ability to upgrade major versions | `bool` | `false` | no |
| rds_family | Maps the engine version with the parameter group family, a family often covers several versions | `string` | `"postgres10"` | no |
| rds_name | Optional name of the RDS cluster. Changing the name will re-create the RDS | `string` | `""` | no |
| replicate_source_db | Specifies that this resource is a Replicate database, and to use this value as the source database. This correlates to the identifier of another Amazon RDS Database to replicate. | `string` | `null` | no |
| skip_final_snapshot | if false(default), a DB snapshot is created before the DB instance is deleted, using the value from final_snapshot_identifier. If true no DBSnapshot is created | `string` | `"false"` | no |
| snapshot_identifier | Specifies whether or not to create this database from a snapshot. This correlates to the snapshot ID you'd find in the RDS console. | `string` | `""` | no |
| storage_type | One of 'standard' (magnetic), 'gp2' (general purpose SSD), 'gp3' (new generation of general purpose SSD), 'io1' (provisioned IOPS SSD), or 'io2' (new generation of provisioned IOPS SSD). If you specify 'io2', you must also include a value for the 'iops' parameter and the `allocated_storage` must be at least 100 GiB (except for SQL Server which the minimum is 20 GiB). | `string` | `"gp3"` | no |
| team_name | Team name | `string` | n/a | yes |
| vpc_name | The name of the vpc (eg.: cloud-platform-live-0) | `string` | n/a | yes |
| vpc_security_group_ids | (Optional) A list of additional VPC security group IDs to associate with the DB instance - in adition to the default VPC security groups granting access from the Cloud Platform | `list(string)` | `[]` | no |

### Outputs

| Name | Description |
|------|-------------|
| database_name | Name of the database |
| database_password | Database Password |
| database_username | Database Username |
| db_identifier | The RDS DB Indentifer |
| irsa_policy_arn | IAM policy ARN for access to create database snapshots |
| rds_instance_address | The hostname of the RDS instance |
| rds_instance_endpoint | The connection endpoint in address:port format |
| rds_instance_port | The database port |
| resource_id | RDS Resource ID - used for performance insights (metrics) |

## Tags

Some of the inputs for this module are tags. All infrastructure resources must be tagged to meet the MOJ Technical Guidance on [Documenting owners of infrastructure](https://technical-guidance.service.justice.gov.uk/documentation/standards/documenting-infrastructure-owners.html).

You should use your namespace variables to populate these. See the [Usage](#how-to-use-this-module) section for more information.

## Reading Material

- [Cloud Platform user guide](https://user-guide.cloud-platform.service.justice.gov.uk/#cloud-platform-user-guide)
- [Amazon RDS for MySQL user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)
- [Amazon RDS for PostgreSQL user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [Amazon RDS for MariaDB user guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MariaDB.html)

I've created a comprehensive, beginner-friendly README that explains the entire project in simple terms. The document is structured to guide someone who isn't familiar with Terraform or AWS through:

## Key Features of the New README:

### 🎯 **Clear Project Overview**
- Explains what the project is in simple terms
- Uses analogies (like "database factory") to make concepts accessible
- Defines technical terms in plain English

### 🏗️ **What Gets Created**
- Breaks down all the AWS resources that are automatically created
- Explains the purpose of each component (security groups, encryption keys, etc.)
- Shows the relationship between different parts

### �� **Database Support**
- Clear table showing supported database types
- Explains when to use each type
- Shows example versions

### 🔧 **Step-by-Step Usage**
- Complete working example with explanations
- Defines what each setting means
- Shows the relationship to business requirements

### 🔐 **Security & Access**
- Explains how credentials work
- Shows how to connect safely
- Emphasizes best practices

### 💰 **Cost Optimization**
- Explains auto start/stop features
- Shows storage autoscaling
- Helps users understand cost implications

### 🛡️ **Security Features**
- Lists what's automatically secured
- Shows additional security options
- Explains why each feature matters

### 📊 **Monitoring & Management**
- Covers performance insights
- Explains backup strategies
- Shows maintenance scheduling

### 🚨 **Important Warnings**
- Emphasizes critical security practices
- Production considerations
- Storage requirements

The README maintains all the technical documentation from the original while adding a comprehensive beginner-friendly section that explains everything in simple terms. It's designed to help someone who's new to cloud infrastructure understand not just how to use the module, but why it's designed the way it is and what problems it solves.
