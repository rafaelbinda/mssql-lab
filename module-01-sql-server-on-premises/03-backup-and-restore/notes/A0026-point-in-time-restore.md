# A0026 – Point-in-Time Restore and Marked Transactions
> **Author:** Rafael Binda  
> **Created:** 2026-04-21  
> **Version:** 2.1  

---

## Descrição  

Este material aborda em profundidade a recuperação de banco de dados até um ponto específico no tempo (Point-in-Time Restore) no SQL Server  
O objetivo é compreender como o SQL Server utiliza o transaction log para reconstruir estados intermediários do banco e como controlar precisamente o ponto final da recuperação utilizando as cláusulas:
- STOPAT  
- STOPATMARK  
- STOPBEFOREMARK  

---

## Hands-on  

[Q0023 - Point-in-Time Restore and Marked Transactions](../scripts/Q0023-sql-point-in-time-restore-and-marked-transactions.sql)  
[INST-Q0015 - Point-in-Time and Marked Transaction Inspection](../../../dba-scripts/SQL-instance-information/INST-Q0015-point-in-time-and-marked-transaction-inspection.sql)

---

## 1 - Conceito de Point-in-Time Restore  

O Point-in-Time Restore permite restaurar um banco de dados para um momento específico no tempo, e não apenas para o final de um backup  
Isso é essencial em cenários como:
- Erro humano (DELETE/UPDATE indevido)  
- Rollback de deploy  
- Auditoria de dados  
- Recuperação seletiva  

---

## 2 - Pré-requisitos  

Para utilizar esse tipo de recuperação, o banco deve atender aos seguintes requisitos:

- Recovery Model:
  - FULL  
  - BULK_LOGGED  

- Backups necessários:
  - FULL  
  - LOG  

### Importante  

No Recovery Model SIMPLE:
- O transaction log é truncado automaticamente  
- Não há histórico suficiente  
- Não é possível realizar restore ponto a ponto  

---

## 3 - Papel do transaction log  

O transaction log é o componente mais importante nesse cenário  

| Tipo de backup | Função |
|---------------|--------|
| FULL | Snapshot do banco |
| DIFFERENTIAL | Alterações desde o FULL |
| LOG | Histórico completo das transações |

### Importante  

Apenas o LOG permite reconstrução granular no tempo  

### Observação  

O backup de LOG deve ser realizado periodicamente  
Sem cadeia de LOG íntegra não é possível realizar recuperação ponto a ponto  

---

## 4 - STOPAT (data/hora)  

Permite restaurar o banco até um momento específico  

### Exemplo  

```sql
RESTORE LOG ExamplesDB
FROM DISK = 'C:\Backups\ExamplesDB.trn'
WITH 
    STOPAT = '2026-04-21 10:30:00',
    RECOVERY;
```

### Comportamento  

- Aplica transações até o horário definido  
- Ignora o restante do log  
- Ideal para erros com horário conhecido  

---

## 5 - Transações marcadas  

Permitem criar pontos de controle no log  

### Exemplo  

```sql
BEGIN TRAN Deploy_V1 WITH MARK 'Deploy version 1';
-- operações
COMMIT;
```

---

## 6 - STOPATMARK  

Recupera o banco até o momento da execução da transação marcada  

```sql
RESTORE LOG ExamplesDB
WITH STOPATMARK = 'Deploy_V1', RECOVERY;
```

### Comportamento  
- Para imediatamente após a execução  
- Ideal para rollback de deploy  

---

## 7 - STOPBEFOREMARK  

Recupera o banco até imediatamente antes da transação marcada  

```sql
RESTORE LOG ExamplesDB
WITH STOPBEFOREMARK = 'Deploy_V1', RECOVERY;
```

### Comportamento  
- Exclui completamente a transação marcada  
- Ideal para evitar aplicação de mudanças  

---

## 8 - Diferença entre STOPAT, STOPATMARK e STOPBEFOREMARK  

| Método | Base | Uso |
|--------|------|-----|
| STOPAT | Tempo | Erro com horário conhecido |
| STOPATMARK | Transação | Deploy controlado |
| STOPBEFOREMARK | Transação | Evitar execução |

---

## 9 - Registro das transações  

As transações marcadas são registradas em `msdb.dbo.logmarkhistory`  

Essa tabela permite:
- Auditoria  
- Troubleshooting  
- Identificação de pontos de restore  

---

## 10 - Ordem correta de restore  

Sempre seguir:

1. Backup do TAIL LOG (quando possível)  
2. FULL (NORECOVERY)  
3. DIFFERENTIAL (NORECOVERY) – se existir  
4. LOGs (NORECOVERY)  
5. Último LOG com STOPAT / STOPATMARK / STOPBEFOREMARK  

### Observação  

- O backup do TAIL LOG deve ser realizado antes do restore quando o banco ainda está acessível  
- Ele captura transações após o último backup de LOG  
- Pode ser a única forma de evitar perda de dados recentes  

---

## 11 - Considerações importantes  

- STOPAT só deve ser aplicado no último LOG  
- Interromper antes invalida sequência  
- Cadeia de LOG deve estar íntegra  
- Sem LOG backup não há recuperação granular  
- STOPAT e STOPATMARK não substituem o TAIL LOG  
- O TAIL LOG garante a captura das últimas transações antes da falha

---

## Referências

- [BEGIN TRANSACTION (Transact-SQL) – WITH MARK](https://learn.microsoft.com/pt-br/sql/t-sql/language-elements/begin-transaction-transact-sql?view=sql-server-ver16)
- [RESTORE Statements - Arguments (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-arguments-transact-sql?view=sql-server-ver16)
- [logmarkhistory (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-tables/logmarkhistory-transact-sql?view=sql-server-ver16)
- [Restaurar um banco de dados do SQL Server até um ponto no tempo](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model?view=sql-server-ver16)

