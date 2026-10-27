# Week 4 Decision Guide: Which AWS Storage or Database Do I Use?

| Service | Use this when... |
|---|---|
| **S3** | You need to store files (images, backups, logs, exports) cheaply and reliably, and you access them by name over the internet or an API, not like a normal disk. |
| **EBS** | You need a hard drive for one EC2 instance, like its operating system or a database running on that server. |
| **EFS** | You need a shared folder that several EC2 instances can read and write at the same time. |
| **RDS** | You need a regular SQL database (MySQL, PostgreSQL) with tables, joins, and transactions, and you want AWS to handle backups and patching. |
| **DynamoDB** | You need a very fast database that scales on its own and you always look data up by a known key, like a listing ID, without needing joins. |

## Quick way to remember

- **Files and backups** → S3
- **One server's disk** → EBS
- **Shared disk across servers** → EFS
- **Relational data and SQL queries** → RDS
- **Fast key lookups at any scale** → DynamoDB