# 01 - SQL Introduction

This module covers the fundamental concepts required to understand and start working with SQL Server, including architecture, data types, querying, transactions and programming objects.

Each topic includes:

- **Notes** → conceptual explanation  
- **Hands-on** → practice scripts inside this module  
- **DBA** → real-world scripts for investigation and troubleshooting  

---

## Study Material

| # | Topic | Note | Hands-on | DBA |
|---|-------|------|----------|-----|
| 1 | Study Environment Setup | [A0001](notes/A0001-study-environment-setup.md) | — | — |
| 2 | Collation | [A0002](notes/A0002-collation.md) | [Q0001](scripts/Q0001-collation.sql) | [INST-Q0001 - Collation](../../dba-scripts/SQL-instance-information/INST-Q0001-collation.sql) |
| 3 | SQL Server Installation | [A0003](notes/A0003-sql_server-installation.md) | — | — |
| 4 | Connectivity Troubleshooting | [A0004](notes/A0004-sql-server-connectivity-troubleshooting.md) | — | [CONN-Q0001 - Active Connections](../../dba-scripts/SQL-connections/CONN-Q0001-active-connections.sql) |
| 5 | Network Commands | [A0005](notes/A0005-network-commands.md) | — | — |
| 6 | SQL Server Version | [A0006](notes/A0006-sql-server-version.md) | — | [INST-Q0002 - Server and Service Name](../../dba-scripts/SQL-instance-information/INST-Q0002-server-and-service-name.sql)<br>[INST-Q0003 - Version and Compatibility](../../dba-scripts/SQL-instance-information/INST-Q0003-version-and-compatibility.sql) |
| 7 | SQL Server Architecture | [A0007](notes/A0007-sql-server-architecture.md) | [Q0002](scripts/Q0002-create-database.sql) | [INST-Q0004 - Physical Storage Layout](../../dba-scripts/SQL-instance-information/INST-Q0004-physical-storage-layout.sql) |
| 8 | SQL Fundamentals | [A0008](notes/A0008-sql-fundamentals.md) | [Q0003](scripts/Q0003-sql-fundamentals.sql) | — |
| 9 | Data Querying | [A0009](notes/A0009-sql-data-querying.md) | [Q0004](scripts/Q0004-sql-data-querying.sql) | — |
|10 | Data Types | [A0010](notes/A0010-sql-data-types.md) | [Q0005](scripts/Q0005-sql-string-data-types.sql)<br>[Q0006](scripts/Q0006-sql-numeric-and-bit-data-types.sql)<br>[Q0007](scripts/Q0007-sql-date-and-time-data-types.sql)<br>[Q0008](scripts/Q0008-sql-special-data-types.sql) | — |
|11 | Transactions & Concurrency | [A0011](notes/A0011-sql-transactions-and-concurrency.md) | [Q0009](scripts/Q0009-sql-transactions-and-concurrency.sql) | [TRAN-Q0001 - Blocking Troubleshooting Queries](../../dba-scripts/SQL-transactions-and-concurrency/TRAN-Q0001-blocking-troubleshooting-queries.sql) |
|12 | Programming Objects | [A0012](notes/A0012-programming-objects-views.md)<br>[A0013](notes/A0013-programming-objects-stored-procedures.md)<br>[A0014](notes/A0014-programming-objects-functions.md)<br>[A0015](notes/A0015-programming-objects-stored-triggers.md) | [Q0010](scripts/Q0010-sql-views.sql)<br>[Q0011](scripts/Q0011-sql-procedures.sql)<br>[Q0012](scripts/Q0012-sql-functions.sql)<br>[Q0013](scripts/Q0013-sql-triggers.sql) | [VIEW-Q0001 - View Metadata Objects](../../dba-scripts/SQL-programming-objects/VIEW-Q0001-view-metadata.sql)<br>[PROC-Q0001 - Stored Procedure Metadata Objects](../../dba-scripts/SQL-programming-objects/PROC-Q0001-procedures-metadata.sql)<br>[FUNC-Q0001 - Function Metadata Objects](../../dba-scripts/SQL-programming-objects/FUNC-Q0001-function-metadata.sql)<br>[TRIG-Q0001 - Trigger Metadata Objects](../../dba-scripts/SQL-programming-objects/TRIG-Q0001-trigger-metadata.sql) |

---

## Tools

Utilities used during environment setup and troubleshooting.

| Tool | Description |
|------|------------|
| [Enable Hyper-V on Windows Home](../tools/A0001_X_EnableHyperV_Windows_Home.bat) | Enables Hyper-V feature on Windows Home |
| [PortQry](../tools/A0003_X_PortQry.zip) | Network port troubleshooting tool |
| [sp_WhoIsActive](../tools/Q0001-sp_whoisactive-v11.32.sql) | SQL Server activity monitoring tool |
