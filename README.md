# terraform-aws-rds

Creates a private RDS instance (Postgres or MySQL), an AWS-managed master
password (Secrets Manager), optional enhanced monitoring, and CloudWatch
alarms for free storage and CPU.

## Security

- `manage_master_user_password = true` — AWS generates, stores, and
  rotates the master password in Secrets Manager. **No credential of any
  kind is ever written to Terraform state or a `.tf` file.**
- `publicly_accessible = false` — reachable only from `rds_sg_id`
  (normally the ECS security group), never from the internet.
- `storage_encrypted = true`; `skip_final_snapshot` is tied to
  `deletion_protection` — protected (prod) instances always take a final
  snapshot on destroy.

## Usage

```hcl
module "rds" {
  source = "../modules/terraform-aws-rds"

  project_name = "hotel-bookings"
  environment  = "dev"
  common_tags  = { Project = "hotel-bookings", Environment = "dev" }

  database_subnet_group_name = module.vpc.database_subnet_group_name
  rds_sg_id                  = module.sg[2].sg_id

  engine          = "postgres"
  engine_version  = "16.4"
  db_name         = "bookings"
  master_username = "app_admin"

  instance_class          = "db.t4g.micro"
  allocated_storage       = 20
  max_allocated_storage   = 100
  multi_az                = false
  backup_retention_period = 3
  deletion_protection     = false
  monitoring_interval     = 0
  alarm_sns_topic_arn     = null
}
```

## Inputs

| Name                        | Type       | Default             | Description                                    |
|--------------------------------|------------|-----------------------|-----------------------------------------------------|
| `project_name` / `environment`     | string      | –                       | Used in resource naming and tags.                       |
| `database_subnet_group_name`         | string      | –                       | DB subnet group (from the VPC module).                     |
| `rds_sg_id`                            | string      | –                       | Security group attached to the instance.                       |
| `engine` / `engine_version`               | string      | `postgres` / `16.4`        | `postgres` or anything else → treated as `mysql`.                 |
| `instance_class`                             | string      | –                       | e.g. `db.t4g.micro`, `db.r6g.large`.                                |
| `allocated_storage` / `max_allocated_storage`   | number      | `20` / `100`               | Storage + autoscaling ceiling (gp3).                                    |
| `multi_az`                                        | bool        | –                       | Enable Multi-AZ (prod).                                                    |
| `backup_retention_period`                            | number      | –                       | Days of automated backups.                                                   |
| `deletion_protection`                                   | bool        | –                       | Also controls whether a final snapshot is taken on destroy.                      |
| `monitoring_interval`                                      | number      | `0`                     | Enhanced monitoring interval in seconds; `0` disables it.                            |
| `alarm_sns_topic_arn`                                         | string      | `null`                  | SNS topic for alarm actions; alarms exist either way.                                    |

## Outputs

| Name                    | Description                                              |
|----------------------------|---------------------------------------------------------------|
| `db_endpoint`                  | `host:port` endpoint (mark sensitive at the caller).             |
| `db_address`                     | Host only, no port.                                                 |
| `db_port`                            | Port number.                                                            |
| `db_instance_id`                       | RDS instance identifier.                                                  |
| `secret_arn`                              | ARN of the AWS-managed master-user secret.                                   |

## Notes

- Fetch the master password at any time with:
  `aws secretsmanager get-secret-value --secret-id <secret_arn>`.
