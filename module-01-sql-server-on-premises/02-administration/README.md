# 02 - Administration

This module covers core SQL Server administration topics, including database architecture, storage, file management, security, and workload control.

Each topic includes:

- **Notes** → conceptual explanation  
- **Hands-on** → practice scripts inside this module  
- **DBA** → real-world scripts for investigation and troubleshooting  

---

## Study Material

| # | Topic | Note | Hands-on | DBA |
|---|------|------|----------|-----|
| 1 | Database Architecture | [A0016](notes/A0016-database-architecture.md) | [Q0002](../01-sql-introduction/scripts/Q0002-create-database.sql) | [INST-Q0004 - Physical Storage Layout](../../dba-scripts/SQL-instance-information/INST-Q0004-physical-storage-layout.sql)<br>[INST-Q0005 - SQL Database Files and Filegroups](../../dba-scripts/SQL-instance-information/INST-Q0005-database-files-and-filegroups-overview.sql) |
| 2 | Database Properties and Access Modes | [A0017](notes/A0017-database-properties-and-access-modes.md) | — | [INST-Q0003 - SQL Version and Compatibility](../../dba-scripts/SQL-instance-information/INST-Q0003-version-and-compatibility.sql)<br>[INST-Q0006 - Database Properties and Access Modes](../../dba-scripts/SQL-instance-information/INST-Q0006-database-properties-and-access-modes.sql) |
| 3 | Database File Management and Maintenance | [A0018](notes/A0018-database-file-management.md) | [Q0014](scripts/Q0014-sql-database-file-management.sql)<br>[Q0015](scripts/Q0015-sql-move-database-file.sql) | [INST-Q0007 - Database Files Space Usage](../../dba-scripts/SQL-instance-information/INST-Q0007-database-files-space-usage.sql)<br>[INST-Q0008 - SQL Virtual Log File Overview](../../dba-scripts/SQL-instance-information/INST-Q0008-log-vlf-overview.sql) |
| 4 | Database Storage and Performance | [A0019](notes/A0019-database-performance-and-storage.md) | [Q0002](../01-sql-introduction/scripts/Q0002-create-database.sql)<br>[Q0009](../01-sql-introduction/scripts/Q0009-sql-transactions-and-concurrency.sql)<br>[Q0016](scripts/Q0016-sql-tempdb-and-file-configuration.sql) | [CONN-Q0001 - Active connections analysis](../../dba-scripts/SQL-connections/CONN-Q0001-active-connections.sql)<br>[INST-Q0005 - Database files and filegroups overview](../../dba-scripts/SQL-instance-information/INST-Q0005-database-files-and-filegroups-overview.sql)<br>[INST-Q0009 - Database I/O and Performance Metrics](../../dba-scripts/SQL-instance-information/INST-Q0009-database-io-and-performance-metrics.sql) |
| 5 | Transparent Data Encryption (TDE) | [A0020](notes/A0020-transparent-data-encryption.md) | [Q0017](scripts/Q0017-sql-transparent-data-encryption.sql) | [INST-Q0010 - Transparent Data Encryption Overview](../../dba-scripts/SQL-instance-information/INST-Q0010-transparent-data-encryption-overview.sql) |
| 6 | Resource Governor (RG) | [A0021](notes/A0021-resource-governor.md) | [Q0018](scripts/Q0018-resource-governor-configuration.sql) | [INST-Q0011 - Resource Governor Overview](../../dba-scripts/SQL-instance-information/INST-Q0011-resource-governor-overview.sql) |
| 7 | Resource Governor and TempDB (Future Study) | [A0022](notes/A0022-resource-governor-tempdb.md) | Pending | Pending |
