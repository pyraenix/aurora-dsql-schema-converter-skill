# Official References & Source Links

All documentation sources used to build and validate this skill's conversion rules.

---

## AWS Official Documentation

### Core DSQL Documentation

| Document | URL | Used For |
|---|---|---|
| Aurora DSQL User Guide (root) | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/ | Overall platform reference |
| Supported Data Types | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-data-types.html | Type mappings, storage sizes, limits |
| Supported SQL Features | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-sql-features.html | DDL/DML/DCL/TCL support matrix |
| PostgreSQL Compatibility | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility.html | Feature compatibility overview |
| Migration Guide | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html | Migration patterns, architectural differences |
| Considerations | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/considerations.html | Operational behaviors, schema grants, connection notes |
| Quotas and Limits | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/CHAP_quotas.html | Hard limits (tables, schemas, rows, connections) |
| Concurrency Control | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-concurrency-control.html | OCC behavior, SQLSTATE 40001, retry patterns |
| Sequences and Identity Columns | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/sequences-identity-columns.html | CACHE requirements, identity column syntax |
| Asynchronous Indexes | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-indexes.html | CREATE INDEX ASYNC, monitoring, limitations |
| CREATE TABLE Syntax | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/create-table-syntax-support.html | Supported CREATE TABLE clauses, STORAGE keyword |
| ALTER TABLE Syntax | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/alter-table-syntax-support.html | Supported ALTER TABLE operations |
| CREATE VIEW | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/create-view-overview.html | View support |
| CREATE SEQUENCE | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/create-sequence-overview.html | Sequence syntax, CACHE rules |
| EXPLAIN Plans | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-explain-plans.html | Query optimization, pushdown behavior |
| System Tables | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-systems-tables.html | sys.jobs, pg_index, catalog queries |

### Authentication & Authorization

| Document | URL | Used For |
|---|---|---|
| Authentication and Authorization | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/authentication-authorization.html | IAM + database role model |
| Database Roles and IAM | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/using-database-and-iam-roles.html | CREATE ROLE, GRANT, custom roles, admin role |
| IAM Policies | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/security_iam_id-based-policy-examples.html | dsql:DbConnect, resource policies |
| Access Control Best Practices | https://aws.amazon.com/blogs/database/securing-amazon-aurora-dsql-access-control-best-practices/ | Security patterns |

### SDKs & Adapters

| Document | URL | Used For |
|---|---|---|
| Aurora DSQL Adapters | https://docs.aws.amazon.com/aurora-dsql/latest/userguide/aws-sdks.html#aurora-dsql-adapters | ORM compatibility, driver support |
| Aurora DSQL Starter Kit | https://awslabs.github.io/aurora-dsql-starter-kit/ | Quick-start patterns, code examples |

---

## AWS Blog Posts

| Title | URL | Published | Used For |
|---|---|---|---|
| DSQL SQL Dialect: How Aurora DSQL differs from single-instance PostgreSQL | https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/ | Apr 2026 | PK-ordered storage, OCC, covering indexes, SELECT FOR UPDATE |
| Up and running with Apache OFBiz and Amazon Aurora DSQL | https://aws.amazon.com/blogs/database/up-and-running-with-apache-ofbiz-and-amazon-aurora-dsql/ | Mar 2025 | OCC retry patterns, auth model differences |
| Amazon Aurora DSQL for global-scale financial transactions | https://aws.amazon.com/blogs/database/amazon-aurora-dsql-for-global-scale-financial-transactions/ | May 2026 | Transaction patterns, idempotency |

---

## PostgreSQL Reference (for source dialect)

| Document | URL | Used For |
|---|---|---|
| PostgreSQL 16 Data Types | https://www.postgresql.org/docs/16/datatype.html | Source type definitions |
| PostgreSQL 16 JSON Functions | https://www.postgresql.org/docs/current/functions-json.html | JSON/JSONB operator compatibility |
| PostgreSQL 16 PL/pgSQL | https://www.postgresql.org/docs/16/plpgsql.html | Trigger/function patterns to transpile |
| PostgreSQL 16 Index Types | https://www.postgresql.org/docs/16/indexes-types.html | GIN, GiST, BRIN source behavior |
| PostgreSQL 16 Wire Protocol | https://www.postgresql.org/docs/current/protocol.html | Driver compatibility basis |

---

## How References Map to Skill Files

| Skill File | Primary Sources |
|---|---|
| SKILL.md | Migration Guide, Supported SQL Features, Considerations |
| dsql-constraints.md | Quotas and Limits, Considerations, Concurrency Control, Database Roles |
| postgresql-type-mappings.md | Supported Data Types |
| plpgsql-patterns.md | Migration Guide (trigger alternatives), PostgreSQL PL/pgSQL docs |
| dsql-function-compatibility.md | Supported SQL Features, JSON Functions (PG docs), Supported Data Types |
| dsql-migration-patterns.md | Concurrency Control, Migration Guide, Quotas and Limits |
| dsql-alter-table-matrix.md | ALTER TABLE Syntax Support, CREATE TABLE Syntax Support |
| dsql-multi-region.md | Multi-Region Starter Kit, Endpoint Routing Blog, Financial Transactions Blog |
| orm-django.md | Aurora DSQL Django Adapter, aurora-dsql-django PyPI, Django Pet Clinic Example |
| orm-hibernate.md | Aurora DSQL Hibernate Adapter, JDBC/HikariCP Sample, Spring Boot Sample, Liquibase Sample |
| orm-rails.md | Rails Sample, Ruby pg Driver Sample, Rails IAM Auth Docs |
| extending-dialects.md | Supported Data Types (NormalizedType definitions) |
| REFERENCES.md | All sources (this file) |

---

## Version Notes

- **DSQL version basis:** PostgreSQL v16 compatible (as of May 2026)
- **Wire protocol:** v3.2+
- **Last verified against official docs:** May 2026
- **Feature set evolves** — always verify against current documentation before production migration

---

## Reporting Inaccuracies

If you find a discrepancy between this skill and the official DSQL documentation:
1. Check the official docs link above for the relevant section
2. The official AWS documentation is always the source of truth
3. Update the relevant reference file in this skill
4. Note the change date and source in a commit message
