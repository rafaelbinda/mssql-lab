# A0029 – Backup Encryption

> **Author:** Rafael Binda  
> **Created:** 2026-04-17  
> **Version:** 1.0  

---

## Descrição

Este material aborda o recurso de criptografia de backup no SQL Server (Backup Encryption), permitindo proteger arquivos de backup (.bak) contra acesso não autorizado

Diferente do Transparent Data Encryption (TDE), que protege os dados em repouso e automaticamente os backups, a criptografia de backup é aplicada explicitamente no momento da execução do backup

Esse recurso é essencial em cenários onde backups são armazenados fora do ambiente principal, transferidos entre servidores ou utilizados para fins de auditoria e compliance

---

## Hands-on

[Q0026 - Backup Encryption](../scripts/Q0026-backup-encryption.sql)  
[INST-Q0018 - Backup Encryption and Certificate Validation](../../../dba-scripts/SQL-instance-information/INST-Q0018-backup-encryption-and-certificate-validation.sql)

---

## Conteúdo relacionado

Para melhor entendimento da criptografia no SQL Server, consultar:

- [A0020 - Transparent Data Encryption (TDE)](../../02-administration/notes/A0020-transparent-data-encryption.md)
- [Q0017 - Transparent Data Encryption (TDE)](../../02-administration/scripts/Q0017-sql-transparent-data-encryption.sql)
- [INST-Q0010 - Transparent Data Encryption (TDE) Overview](../../../dba-scripts/SQL-instance-information/INST-Q0010-transparent-data-encryption-overview.sql)

---

## 1 - O que é Backup Encryption

Backup Encryption é o processo de criptografar um backup no momento da sua criação

### Objetivo

- Proteger arquivos de backup (.bak)
- Evitar acesso indevido aos dados
- Garantir segurança em transporte e armazenamento

---

## 2 - Vantagem principal

Se alguém tiver acesso ao arquivo de backup, não conseguirá restaurar o banco sem acesso ao certificado ou chave de criptografia utilizada

---

## 3 - Diferença em relação ao TDE

| Característica | TDE | Backup Encryption |
|--------------|-----|------------------|
| Criptografa dados | ✔️ | ❌ |
| Criptografa backup | ✔️ | ✔️ |
| Automático | ✔️ | ❌ |
| Necessita configuração no backup | ❌ | ✔️ |

---

## 4 - Pré-requisitos

Antes de realizar um backup criptografado, é necessário:

- Criar Database Master Key (DMK) no banco master
- Criar certificado ou chave assimétrica

---

## 5 - Backup criptografado

Exemplo conceitual:

```sql
BACKUP DATABASE ExamplesDB_TDE
TO DISK = 'C:\Temp\ExamplesDB_TDE_ENCRYPTED.bak'
WITH 
    ENCRYPTION 
    (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupCert
    );
```

### Observações

- A criptografia NÃO é automática
- Deve ser declarada explicitamente no backup
- Utilizar AES_256 é recomendado

---

## 6 - Restore com backup criptografado

### Cenário 1 - Certificado já existe

- Restore executado normalmente

### Cenário 2 - Certificado não existe

Necessário:

- Criar DMK no master (se não existir)
- Importar certificado
- Executar restore normalmente

```sql
-- Criar master key (se necessário)
USE master;
GO

CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword_2026!';

-- Importar certificado
CREATE CERTIFICATE BackupCert
FROM FILE = 'C:\Temp\BackupCert.cer'
WITH PRIVATE KEY (
    FILE = 'C:\Temp\BackupCert.key',
    DECRYPTION BY PASSWORD = 'StrongPassword_2026!'
);
```

---

## 7 - Erro comum

```text
Msg 33111
Cannot find server certificate with thumbprint...
```

### Causa

O SQL Server não encontra o certificado utilizado na criptografia do backup

### Solução

- Importar o certificado correto
- Garantir acesso à chave privada

---

## 8 - Considerações operacionais

- Aumenta a segurança do backup
- Pode gerar pequeno overhead de CPU
- Não substitui outras camadas de segurança

---

## 9 - Boas práticas

- Backup imediato do certificado
- Armazenar .cer e .key com segurança
- Testar restore
- Utilizar AES_256
- Documentar processo

---

## Referências

- [Backup encryption](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-encryption?view=sql-server-ver16)
- [Create an encrypted backup](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/create-an-encrypted-backup?view=sql-server-ver16)
- [BACKUP (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/backup-transact-sql?view=sql-server-ver16)
- [BACKUP CERTIFICATE (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/backup-certificate-transact-sql?view=sql-server-ver16)
