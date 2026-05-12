# podyn — RepoDocs
_Generated on 2026-05-11_

I have enough context. Now producing the markdown output.

## Summary

### Overview
Podyn is a Java command-line tool that replicates AWS DynamoDB tables to PostgreSQL (optionally distributed via Citus). It performs three operations — schema replication, bulk data load via DynamoDB Scan + Postgres `COPY`, and continuous change replication via DynamoDB Streams (consumed through the Kinesis Client Library). It is a forked copy of the upstream `citusdata/podyn` project, retained inside the PowerReviews / Syndigo org as a standalone migration / data-movement utility (likely used for DynamoDB → Postgres/Citus migrations and ongoing replication scenarios).

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | Java | 1.8 (source/target) |
| Framework | None (CLI app); Apache Commons CLI for argument parsing | commons-cli 1.4 |
| Database | PostgreSQL (JDBC driver) | postgresql 42.1.1 |
| Build Tool | Apache Maven (with maven-shade-plugin for fat JAR) | maven-compiler 3.6.1 / shade 3.0.0 |
| CI/CD | GitHub Actions (secrets-scan workflow only) | n/a |
| Cloud/Infra | AWS SDK for Java — DynamoDB | aws-java-sdk-dynamodb 1.11.1034 |
| Cloud/Infra | Amazon Kinesis Client Library | 1.7.6 |
| Cloud/Infra | DynamoDB Streams Kinesis Adapter | 1.2.2 |
| Library | Google Guava (RateLimiter, etc.) | 22.0 |
| Library | Log4j 2 / SLF4J | log4j 2.19.0 / slf4j 1.7.36 |
| Library | Project Lombok | 1.18.24 |

### Consumers
Podyn is a standalone CLI binary (`./podyn` → `java -jar target/podyn-1.0.jar`); it is invoked by an operator, not imported as a library. No org-internal repo references this artifact (no `groupId` published elsewhere, no Docker image, no Jenkins pipeline file).

| Consumer | Type | How They Use It |
|----------|------|----------------|
| Human operator | CLI user | Runs `./podyn` with flags (`--schema`, `--data`, `--changes`, `--postgres-jdbc-url`, `--citus`) to migrate or continuously replicate DynamoDB tables into PostgreSQL/Citus |
| AWS DynamoDB | External system (source) | Source of `ListTables`/`DescribeTable`/`Scan` and Streams data |
| PostgreSQL / Citus | External system (target) | Destination for `CREATE TABLE`, `COPY ... FROM STDIN`, `INSERT ... ON CONFLICT`, `DELETE`, and `create_distributed_table(...)` |
| GitHub Actions (scheduled secret scan) | CI | Runs trufflehog on a weekday cron via `.github/workflows/secrets-scan.yml` |

### Dependencies on Org Repos
_Not determinable from code._ No org-internal Maven coordinates, git submodules, or imports of other PowerReviews repos are present; all dependencies are public Maven artifacts.

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| AWS DynamoDB | Source of table metadata, scan-based bulk read, and change stream | SDK (`aws-java-sdk-dynamodb`) |
| AWS DynamoDB Streams | Change-data-capture feed (consumed via Kinesis adapter) | SDK (`dynamodb-streams-kinesis-adapter` + `amazon-kinesis-client`) |
| AWS CloudWatch | Required by Kinesis Client Library Worker for metrics | SDK (`AmazonCloudWatchClientBuilder`) |
| AWS STS / default credential chain | Authentication to AWS services | SDK (`DefaultAWSCredentialsProviderChain`) |
| PostgreSQL / Citus | Replication target (DDL, `COPY`, upsert, delete, `create_distributed_table`) | JDBC driver |
| Slack (GitHub Actions only) | Notify `#github-token-scan` on trufflehog failure via `SLACK_WEBHOOK` | Webhook (outbound) |
| trufflehog (GitHub Action) | Scheduled secret scanning | SDK (GitHub Action) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| DynamoDB Streams (per table) | Stream / CDC consumer (KCL Worker, `InitialPositionInStream.TRIM_HORIZON`) | Consumes | Continuous change replication when `--changes` is passed; `processRecords` translates `INSERT`/`MODIFY` → upsert and `REMOVE` → delete |
| KCL lease tables (`podyn_migration_<tableName>`) | DynamoDB checkpoint tables | Both | Persist Kinesis Client Library shard leases / checkpoints so replication can resume after restart |
| Parallel DynamoDB Scan workers | In-process thread pool (`Executors.newCachedThreadPool`) using `TotalSegments`/`Segment` | Consumes | Bulk initial load with `-dw/--num-data-workers`; rate-limited by Guava `RateLimiter` using `-r/--scan-rate` |
| HashedMultiEmitter connection pool | In-process fan-out (N JDBC connections, hashed by distribution key) | Produces | Concurrent upserts/deletes against PostgreSQL while preserving per-key ordering |
| GitHub Actions `secret-scan` | Scheduled CI job (cron `0 14 * * 1-5`) | N/A (for jobs) | Daily weekday trufflehog scan with Slack alert on failure |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| Java (source/target) | 1.8 | Java 8 is past public end-of-public-updates; no LTS support without commercial subscription | Severe (EOL runtime) |
| AWS SDK for Java | v1 (`aws-java-sdk-dynamodb` 1.11.1034) | AWS SDK for Java v1 entered maintenance and end-of-support is announced; v2 is the supported line | Severe (EOL major version) |
| Amazon Kinesis Client Library | 1.7.6 | KCL 1.x is the legacy major version (KCL 2.x is current) | Severe (EOL major version) |
| PostgreSQL JDBC driver | 42.1.1 | Multiple CVEs published against `pgjdbc < 42.x` patch levels (e.g. CVE-2022-21724, CVE-2022-26520, CVE-2024-1597 affect <42.7.2) | Critical (CVEs) |
| Google Guava | 22.0 | Affected by CVE-2018-10237 (unbounded memory allocation) and CVE-2023-2976 (temp-file information disclosure); fixed in 24.1.1 / 32.0.0 respectively | Critical (CVEs) |
| commons-cli | 1.4 | Reached end of its line; superseded; no recent security fixes against this version line | Severe |
| log4j (transitive include filter in shade plugin: `log4j:log4j`) | unspecified | Shade plugin explicitly preserves `log4j:log4j` (Log4j 1.x) artifact classes if present transitively — Log4j 1.x is EOL and has CVE-2019-17571, CVE-2022-23305, CVE-2022-23307 | Critical (CVEs if Log4j 1.x is actually pulled in) |

## API Reference

This is a CLI tool, not a service. The "API" is the command-line interface plus the internal `TableEmitter` SPI.

### CLI — `com.citusdata.migration.DynamoDBReplicator`
Defined in `src/main/java/com/citusdata/migration/DynamoDBReplicator.java`.

| Short | Long | Arg | Default | Purpose |
|-------|------|-----|---------|---------|
| `-h` | `--help` | – | – | Print help |
| `-t` | `--table` | comma-separated names | all non-`podyn_migration_*` tables | DynamoDB table(s) to replicate |
| `-u` | `--postgres-jdbc-url` | JDBC URL | – (stdout) | Destination Postgres; if omitted, SQL is printed to stdout |
| `-s` | `--schema` | – | false | Replicate table schema (CREATE TABLE / CREATE INDEX / `create_distributed_table`) |
| `-d` | `--data` | – | false | Replicate current data via Scan + `COPY` |
| `-c` | `--changes` | – | false | Continuously replicate changes via DynamoDB Streams (KCL Worker) |
| `-x` | `--citus` | – | false | Use Citus `create_distributed_table` with the DynamoDB partition key as distribution column |
| `-m` | `--conversion-mode` | `columns` \| `jsonb` | `columns` | Top-level keys → columns, or store whole item in `data jsonb` |
| `-lc` | `--lower-case-column-names` | – | false | Lowercase column names |
| `-n` | `--num-connections` | int | 16 | JDBC connection pool size for `HashedMultiEmitter` |
| `-r` | `--scan-rate` | int | 25 | Maximum reads/sec during scan (Guava `RateLimiter`) |
| `-dw` | `--num-data-workers` | int | 1 | Parallel-Scan workers (sets DynamoDB `TotalSegments`/`Segment`) |
| `-sl` | `--scan-limit` | int | 100 (min clamped to 50) | Items per individual Scan request |

Exit codes: `0` success/help; `1` table state error, execution error, or generic exception; `2` runtime exception; `3` argument parse error.

### Public Java SPI — `com.citusdata.migration.datamodel.TableEmitter`
```java
TableSchema fetchSchema(String tableName) throws EmissionException;
void createTable(TableSchema tableSchema) throws EmissionException;
void createColumn(TableColumn column) throws EmissionException;
long copyFromReader(TableSchema tableSchema, Reader reader) throws EmissionException;
void upsert(TableRow tableRow) throws EmissionException;
void delete(PrimaryKeyValue primaryKeyValue) throws EmissionException;
void close() throws EmissionException;
```
Implementations: `JDBCTableEmitter` (real Postgres), `StdoutSQLEmitter` (dry-run prints SQL), `HashedMultiEmitter` (fan-out over N emitters, hashed by `TableSchema.distributionColumn`; DDL/`COPY` take a write lock, upsert/delete take read lock and per-emitter `synchronized`).

### Key class — `DynamoDBTableReplicator`
- `replicateSchema()` — `DescribeTable` → builds `TableSchema` (attributes, primary key, GSIs); in `jsonb` mode adds a `data jsonb` column; emits via `emitter.createTable`.
- `startReplicatingData(maxScanRate, numDataWorkers, scanLimitSetting): List<Future<Long>>` — submits N `replicateData` callables to the shared `ExecutorService`.
- `replicateData(...)` — paginated parallel `Scan` (`ConsistentRead=true`, `ReturnConsumedCapacity=TOTAL`, `Limit=scanLimit`, `TotalSegments`/`Segment` when workers > 1), retries `ProvisionedThroughputExceededException`/`InternalServerErrorException` up to 3 times with a 1 s sleep, rate-limits using consumed capacity, batches into `TableRowBatch` and emits via `COPY FROM STDIN`.
- `startReplicatingChanges()` — configures a KCL `Worker` against the table's `LatestStreamArn` using `AmazonDynamoDBStreamsAdapterClient`, lease table `podyn_migration_<tableName>`, `failoverTimeMillis=20000`, `maxRecords=1000`, `idleTimeBetweenReadsInMillis=500`.
- `processRecords(...)` — for each stream record: `INSERT`/`MODIFY` → `emitter.upsert(rowFromDynamoRecord(newImage))`, `REMOVE` → `emitter.delete(primaryKeyValueFromDynamoKeys(keys))`.
- `addNewColumns(item)` — in `columns` mode, alters the Postgres table to add unseen keys; type conflicts produce a suffixed column (e.g. `ip_boolean`).
- `static columnValueFromDynamoValue(AttributeValue)` / `static columnTypeFromDynamoValue(AttributeValue)` — DynamoDB → Postgres type mapping (S→text, N→numeric, B→bytea, BOOL→boolean, M/L/SS/NS/BS→jsonb).

### SQL operations produced
- `CREATE TABLE <name> ( <col type [NOT NULL]>, ..., PRIMARY KEY(...) )`
- `CREATE INDEX "<gsiName>" ON <name>(<cols>)`
- `SELECT create_distributed_table('<name>', '<partitionKey>')` (Citus only)
- `ALTER TABLE <name> ADD COLUMN <col> <type>`
- `COPY <name> FROM STDIN`
- `INSERT INTO <name> (...) VALUES (...) ON CONFLICT (<pk>) DO UPDATE SET ... = EXCLUDED....`
- `DELETE FROM <name> WHERE <pk> = ?::type AND ...`
- Introspection: queries against `information_schema.columns`, `information_schema.table_constraints`, `pg_extension`, `pg_dist_partition`.

### Compatibility shim — `com.citusdata.migration.compatibility.PostgresDynamoDB`
Implements the AWS `AmazonDynamoDB` interface backed by a Postgres `Connection`; nearly every method throws `UnsupportedOperationException` (appears to be an experimental / unfinished facade — not wired into the main flow).

## Architecture

### System context
```
                 ┌────────────────────────┐
                 │ Operator CLI (./podyn) │
                 └───────────┬────────────┘
                             │ args
                             ▼
              ┌──────────────────────────────────┐
              │   DynamoDBReplicator (main)      │
              │   - parses CLI                   │
              │   - builds executor + emitter    │
              │   - creates one Replicator/table │
              └─────┬───────────────────┬────────┘
                    │                   │
                    ▼                   ▼
   ┌─────────────────────────┐  ┌─────────────────────────────┐
   │ DynamoDBTableReplicator │  │     TableEmitter            │
   │ - DescribeTable         │  │  ┌─ JDBCTableEmitter (real) │
   │ - parallel Scan workers │  │  ├─ StdoutSQLEmitter (DRY)  │
   │ - KCL Worker (Streams)  │  │  └─ HashedMultiEmitter      │
   └─────┬─────────────┬─────┘  │     (fan-out, N emitters)   │
         │             │        └────────────┬────────────────┘
         │ AWS SDK v1  │                     │ JDBC
         ▼             ▼                     ▼
   ┌─────────────────────────┐         ┌──────────────────────┐
   │ DynamoDB / Streams /    │         │ PostgreSQL / Citus   │
   │ CloudWatch              │         │  + podyn_migration_* │
   │ (KCL lease tables)      │         │  lease tables stay   │
   └─────────────────────────┘         │  in DynamoDB         │
                                       └──────────────────────┘
```

### Key components
- **`DynamoDBReplicator`** — `main`, CLI parsing, AWS client construction (`DefaultAWSCredentialsProviderChain`), emitter wiring, executor lifecycle, shutdown hook closing emitters.
- **`DynamoDBTableReplicator`** — per-table orchestrator covering schema derivation, parallel scan, KCL worker setup, and stream record translation.
- **`TableSchema` / `TableColumn` / `TableRow` / `TableIndex` / `PrimaryKeyValue`** (`datamodel/`) — produce Postgres DDL/DML strings, handle identifier quoting against `PostgresKeywords`, manage primary keys and the Citus distribution column.
- **`TableEmitter` SPI** with three implementations: stdout dry-run, real JDBC, and a hash-fan-out wrapper that serializes DDL/`COPY` and parallelizes upsert/delete using `(hash(distributionKey) % N)` to pick the connection.
- **`ConversionMode`** enum (`columns` vs `jsonb`) — switches between expanding DynamoDB keys into Postgres columns or storing the whole item in a `data jsonb` column.

### Data flow
1. CLI parsed → AWS credentials resolved via default chain.
2. If a JDBC URL is given, build N `JDBCTableEmitter`s into a `HashedMultiEmitter`; otherwise use `StdoutSQLEmitter`.
3. Determine tables: explicit `--table` list, or `dynamoDBClient.listTables()` filtered to exclude `podyn_migration_*` lease tables.
4. **Schema phase** (`-s`): `DescribeTable` → `TableSchema` → emitter `createTable` (DDL + optional Citus distribution).
5. **Data phase** (`-d`): N workers run parallel segmented Scans, rate-limited by Guava `RateLimiter` based on consumed capacity, batched into `TableRowBatch.asCopyReader()`, loaded via Postgres `COPY FROM STDIN`. New keys trigger `ALTER TABLE ADD COLUMN`.
6. **Changes phase** (`-c`): a KCL Worker reads the table's DynamoDB Stream via the Kinesis adapter, checkpointing into a per-table lease table named `podyn_migration_<tableName>` (stored in DynamoDB, not Postgres). Records → upsert / delete on the emitter.

### CI/CD tooling
**GitHub Actions** (detected via `.github/workflows/secrets-scan.yml`). The only workflow runs trufflehog on a cron `0 14 * * 1-5` and Slack-notifies on failure. There is **no build/test/deploy pipeline in this repo** — releases are produced by running `mvn package` locally per the README; the resulting shaded JAR lives at `target/podyn-1.0.jar`.

### Test architecture
_None present._ No `src/test/` directory, no test framework declared in `pom.xml`.

### Data model / database schema
Schema is dynamic and derived at runtime from DynamoDB. Per the README and `DynamoDBTableReplicator.columnTypeFromDynamoValue`, the DynamoDB → Postgres type mapping is:

| DynamoDB | Postgres |
|----------|----------|
| S (String) | `text` |
| N (Numeric) | `numeric` |
| B (Binary) | `bytea` |
| BOOL | `boolean` |
| NULL | `text` |
| SS / NS / BS / L / M | `jsonb` |

Primary key = DynamoDB hash + range keys; GSIs become Postgres `CREATE INDEX`. When `--citus` is set, the DynamoDB partition (HASH) key becomes the distribution column.

### Auth & trust boundaries
- **Outbound to AWS**: `DefaultAWSCredentialsProviderChain` (env vars / shared credentials file / IAM role).
- **Outbound to Postgres**: credentials embedded in the `--postgres-jdbc-url` JDBC URL (`user`, `password`, `sslmode`).
- **Inbound**: none — there are no listening sockets or HTTP endpoints. The tool runs as an operator-launched CLI.
- **Authorization model**: relies entirely on the IAM principal used by the AWS SDK (must allow `DescribeTable`, `Scan`, `ListTables`, `ListStreams`, `DescribeStream`, `GetShardIterator`, `GetRecords`, plus DynamoDB read/write to the KCL lease table) and the Postgres role used in the JDBC URL.

### Data ownership
| Datastore | Access pattern | Entities owned |
|-----------|----------------|----------------|
| PostgreSQL / Citus | **Owner** (creates tables, adds columns, COPY-loads, upserts, deletes) | One table per replicated DynamoDB table, named identically; optional `data jsonb` column in jsonb mode |
| DynamoDB (source tables) | **Reader** | None — read via Scan and Streams |
| DynamoDB (lease tables) | **Owner** | `podyn_migration_<tableName>` tables (KCL checkpointing) |

No shared-datastore signal: connection URLs are passed at runtime; no fixed database name or shared DSN that could indicate co-ownership with another sibling repo.

### Deployment topology
_Deployment topology not in this repo._ No Dockerfile, Helm chart, Terraform, or k8s manifest. The repo ships a shaded fat JAR and a `./podyn` shell wrapper (`java -jar target/podyn-1.0.jar $*`) intended to be run manually on an operator's host with AWS credentials configured.

## Repo Activity
Derived from git history; current HEAD is `e5502a4`.

- **Created**: 2017-06-26 (`46bb439 Initial commit` by Marco Slot at Citus Data; fork imported into PowerReviews org).
- **Last meaningful change**: 2022-11-15 — `8ac877c enhancement: make scan-limit configurable to faciliate higher throughput potential.` (`6304f7e` adding parallel scan landed the same day). The most recent commit (`e5502a4 Update README.md`, 2022-11-15) is a docs-only update.
- **Activity level**: 0 commits in the last 90 days. The repo has been effectively dormant since 2022-11-15 — roughly 3.5 years of no commits at the time of this analysis (2026-05-12).
- **Hot spots** (since the PowerReviews fork went active in 2022, given the 6-month window is empty):
  1. `src/main/java/com/citusdata/migration/DynamoDBTableReplicator.java` — 3 touches (parallel scan, configurable scan-limit, logging conversion).
  2. `src/main/java/com/citusdata/migration/DynamoDBReplicator.java` — 3 touches (new CLI options for `--num-data-workers` and `--scan-limit`, slf4j logging).
  3. `pom.xml` — 2 touches (Log4j2/SLF4J + Lombok updates).
- **Recent major changes**: _No major changes in the last 6 months._ Last meaningful work was the November 2022 batch: PRs #1/#2 migrated logging from Log4j 1.x to Log4j 2 / SLF4J with Lombok `@Slf4j`; PR #3 added parallel DynamoDB Scan via `TotalSegments`/`Segment` controlled by `--num-data-workers`; PR #4 made the per-scan item `--scan-limit` configurable. No source changes have landed since.
