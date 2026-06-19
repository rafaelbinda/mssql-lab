# A0023 – Backup Fundamentals

> **Author:** Rafael Binda  
> **Created:** 2026-04-08  
> **Version:** 4.0  

---

## Descrição

Este material apresenta os conceitos fundamentais de backup no SQL Server, incluindo tipos de backup, funcionamento interno, dependências entre backups, estratégias de armazenamento e conceitos críticos como RPO, RTO e cadeia de backup

---

## Hands-on

[Q0019 - Backup FULL / DIFFERENTIAL / LOG](../scripts/Q0019-sql-backup-full-differential-log.sql)  
[INST-Q0016 - Backup Media and History Analysis](../../../dba-scripts/SQL-instance-information/INST-Q0016-backup-media-and-history-analysis.sql)

---

## 1 - Visão geral de backup

O SQL Server permite diferentes tipos de backup que podem ser combinados para atender requisitos de negócio

- Backup é online (não bloqueia uso do banco)
- Pode ser executado durante operações normais
- Tipos de backup são complementares

---

## 2 - Planejamento de backup (RPO e RTO)

**A definição de RPO e RTO não é responsabilidade exclusiva do DBA**, ela deve ser feita em conjunto com:
- Gestor da aplicação
- Áreas de negócio

### Linha do tempo

```
Tempo -------------------------------------------------------------->

Backup realizado        Falha ocorre                Sistema restaurado
       |                     |                              |
       |------ RPO ---------|<----------- RTO ------------->|
```

### RPO - Recovery Point Objective
- Intervalo entre o último backup e a falha
- Representa a quantidade de dados que pode ser perdida
- Devemos perguntar quantos dados é aceitável perder?

### RTO - Recovery Time Objective
- Tempo máximo para recuperação
- Representa o tempo de indisponibilidade do sistema
- Devemos perguntar quanto tempo podemos aguardar para que o restore seja finalizado?

---

## 3 - Requisitos de armazenamento e operação

Uma estratégia de backup eficiente não se resume à execução dos comandos de backup  
Ela envolve planejamento de armazenamento, validação contínua e processos operacionais bem definidos  
A ausência desses controles pode comprometer completamente a recuperação do ambiente em caso de falha

### Armazenamento

O armazenamento deve ser planejado considerando desempenho, custo e confiabilidade

- Avaliar onde os backups serão armazenados:
  - Disco local
  - Storage dedicado
  - Nuvem

- Considerar custos envolvidos:
  - Armazenamento (volume de dados)
  - Transferência de dados
    - Egress em cloud: para enviar não tem custo mas para download é cobrado tráfego de saída
- Evitar manter backups apenas no mesmo servidor do banco
  - Falha física pode comprometer banco e backup simultaneamente

- Sempre que possível:
  - Manter cópias em locais distintos (offsite)
  - Utilizar estratégias de redundância

### Política de retenção

A retenção define por quanto tempo os backups serão mantidos antes de serem descartados
- Deve ser maior que a frequência de execução do DBCC CHECKDB
- O DBCC CHECKDB identifica corrupção lógica e física no banco
- Caso a retenção seja menor, existe o risco de todos os backups disponíveis já estarem corrompidos

#### Exemplo prático

- DBCC CHECKDB executado a cada 7 dias
- Retenção configurada para 5 dias

Se ocorrer uma corrupção logo após a execução do CHECKDB, ela só será detectada na próxima execução  
Nesse momento, todos os backups válidos podem já ter sido sobrescritos

### Testes de restore

Backup sem teste de restore não garante recuperação

- Deve existir um processo periódico de validação de restore
- Testes devem incluir:
  - Restore completo
  - Restore com diferencial
  - Restore com log (quando aplicável)

- Validar:
  - Integridade do backup
  - Tempo de recuperação (RTO)
  - Consistência dos dados restaurados

### Monitoramento

A execução dos backups deve ser monitorada continuamente

- Validar sucesso das rotinas de backup
- Configurar alertas para falhas
- Verificar:
  - Tamanho dos backups
  - Tempo de execução
  - Frequência esperada

Falhas silenciosas de backup são uma das principais causas de perda de dados em ambientes de produção

### Versionamento e dependências

Backups de banco de dados não substituem controle de versão de aplicações

- Mudanças em estrutura (DDL) devem ser versionadas
- Scripts de deploy devem ser controlados
- Dependências entre banco e aplicação devem ser consideradas

### Ambiente de testes

- Manter ambiente dedicado para testes de restore
- Simular cenários reais de falha
- Validar procedimentos operacionais

Esse processo garante que, em caso de incidente, a recuperação será executada de forma previsível

*Sem testes, monitoramento e retenção adequada, o backup deixa de ser uma solução confiável*

---

## 4 - Backup full

- Backup completo do banco de dados
- Leva todo o conteúdo dos arquivos de dados `MDF` e `LDF`
- Leva do arquivo de log apenas a atividade durante o backup full

```
    Tempo -------------------------------------------------------------->
    22:00 --------------------------- 23:00
    Início do backup                  Fim do backup
```

Ponto de recuperação: 23:00

### Funcionamento interno do backup (consistência)

Durante a execução de um backup, o SQL Server utiliza o transaction log para garantir a consistência dos dados

O processo ocorre da seguinte forma:

1. No início do backup, o SQL Server executa um CHECKPOINT
   - Isso garante que as páginas sujas (dirty pages) sejam gravadas em disco
   - Nesse momento é registrado um LSN (Log Sequence Number) de referência

2. Durante o backup, os arquivos de dados são copiados

3. Ao final do processo, o SQL Server consulta o transaction log
   - A partir do LSN inicial capturado no CHECKPOINT
   - Incluindo todas as alterações até o término do backup

4. No momento do restore:
   - Todas as transações finalizadas (commitadas) são aplicadas
   - Transações que estavam em andamento são desfeitas (rollback)

Isso garante que o banco seja restaurado exatamente no estado consistente do momento em que o backup foi concluído

---

## 5 - Backup diferencial

- Backup dos dados alterados desde o último backup full
- Precisa de um backup full para ser recuperado na sequencia
- Leva do arquivo de log apenas a atividade durante o backup full

```
    Tempo -------------------------------------------------------------->
    20:00        21:00        22:00        23:00
    FULL         → DIF1      → DIF2       → DIF3
```

Restore necessário:
FULL + último DIF (DIF3)

- Não é possível fazer restore só de 1 diferencial
Exemplo: FULL + DIF2

### Funcionamento interno do backup diferencial (consistência)

O backup diferencial utiliza o mesmo mecanismo de consistência baseado no transaction log, porém com foco apenas nas alterações ocorridas desde o último backup full  
O processo ocorre da seguinte forma:

1. Após a execução de um backup full, o SQL Server inicializa um conjunto de páginas internas de controle (mapa de extents modificadas)
   - Esse mapa registra quais extents sofreram alterações
   - A partir desse momento, qualquer modificação em uma página marca a extent correspondente como alterada

2. No início do backup diferencial, o SQL Server executa um CHECKPOINT
   - Isso garante a consistência das páginas em disco
   - Um LSN (Log Sequence Number) de referência é capturado

3. Durante o backup diferencial:
   - O SQL Server consulta o mapa de extents modificadas
   - Todas as extents alteradas desde o último backup full são copiadas para o backup

4. Ao final do processo:
   - O SQL Server lê o transaction log a partir do LSN capturado no CHECKPOINT
   - Inclui todas as alterações até o término do backup

5. No momento do restore:
   - O backup full é restaurado
   - Em seguida, o backup diferencial mais recente é aplicado
   - Todas as transações finalizadas são aplicadas
   - Transações em andamento são desfeitas (rollback)

Isso garante que o banco seja recuperado exatamente no estado consistente do momento em que o backup diferencial foi concluído

---

## 6 - Backup de log

- Não pode estar no modo RECOVERY MODEL SIMPLE
- Realiza o backup de todo o conteúdo do transaction log (ldf)
- Precisa de um backup full para ser recuperado em sequencia
- No final do backup trunca o log (apaga a porção inativa)

```
    Tempo -------------------------------------------------------------->
    20:00        21:00        22:00        23:00
    FULL        → LOG1       → LOG2       → LOG3
```

Restore:
FULL + LOG1 + LOG2 + LOG3 (todos os LOGs em sequência)

- Um backup do log é dependente do outro
- Ocorre um controle do SQL Server pelo LSN incial e LSN final

---

## 7 - Backup COPY_ONLY

O backup COPY_ONLY é um tipo especial de backup que não interfere na cadeia de backups existente  
Ele é utilizado quando é necessário realizar um backup pontual sem impactar a estratégia padrão configurada no ambiente

### Objetivo

Permitir a criação de backups sob demanda sem alterar:
- A base de backups diferenciais
- A sequência de backups de log

### Funcionamento

O comportamento varia conforme o tipo de backup:

#### COPY_ONLY FULL

- Não altera a base para backups diferenciais
- O próximo backup diferencial continua baseado no último FULL tradicional

#### COPY_ONLY LOG

- Não interfere na sequência da cadeia de backups de log
- Mantém a continuidade dos LSNs

### Exemplo prático

Cenário:
- Backup FULL diário às 00:00
- Backups diferenciais ao longo do dia

Durante o dia, um backup manual é executado:
`BACKUP DATABASE MinhaBase TO DISK = 'backup.bak' WITH COPY_ONLY;`

Resultado:
- Esse backup não passa a ser a nova base do diferencial
- O próximo diferencial continua baseado no FULL das 00:00

### Quando utilizar

- Antes de testes ou intervenções no ambiente
- Para envio de backup para outro ambiente
- Para cópias temporárias de segurança
- Em atividades de troubleshooting

### Quando NÃO utilizar

- Como estratégia padrão de backup
- Substituindo backups FULL regulares

---

## 8 - Tail log backup (NO_TRUNCATE)

O Tail Log Backup é o backup final do transaction log realizado após uma falha, com o objetivo de capturar todas as transações ocorridas desde o último backup de log

- Ele é utilizado para evitar perda de dados em cenários críticos
- É a **ultima tentativa de recuperar dados** após falha crítica
- É a **última oportunidade de preservar dados** antes da recuperação do banco
- Esse backup deve ser a **primeira ação** sempre que houver possibilidade de recuperar o log
- O Tail Log Backup não substitui backups regulares
- Ele é uma medida emergencial
- Deve fazer parte do procedimento padrão de recuperação de desastres
- Sua execução pode ser a diferença entre uma recuperação completa e perda parcial de dados
- O Tail Log Backup deve ser tratado como um reflexo operacional do DBA em cenários de falha

### Objetivo

Capturar a parte final do log (tail of the log), garantindo que todas as transações recentes sejam preservadas antes do processo de restore

### Analogia - Reflexo medular

- O Tail Log Backup pode ser comparado a um reflexo medular
- Assim como no reflexo da patela, onde o corpo reage automaticamente sem passar pelo cérebro, ao detectar um banco em estado SUSPECT, o DBA deve agir imediatamente executando o Tail Log Backup
- Essa ação deve ser rápida e automática, com o objetivo de preservar o máximo possível de transações antes de qualquer outra intervenção

### Conceito aplicado

- Banco SUSPECT → situação crítica
- Primeira ação → BACKUP LOG com NO_TRUNCATE
- Objetivo → preservar dados recentes

### Quando utilizar

- Banco em estado SUSPECT
- Falha nos arquivos de dados (MDF/NDF)
- Falha inesperada do banco
- Antes de iniciar um processo de restore

### Funcionamento

- O SQL Server copia todo o conteúdo restante do transaction log
- Inclui transações ainda não protegidas por backups anteriores
- Não realiza truncamento do log durante o processo
- Esse comportamento garante a preservação máxima dos dados disponíveis

### Exemplo prático

- Último backup de log realizado às 22:00
- Falha ocorre às 23:00

Existe um intervalo de 1 hora de transações que ainda não foram salvas em backup

Sem o Tail Log Backup:
- Essas transações serão perdidas

Com o Tail Log Backup:
- É possível recuperar o banco até o momento exato da falha

### Requisitos

- Banco deve estar em recovery model FULL ou BULK_LOGGED
- O arquivo de log deve estar íntegro

### Limitações

- Se o arquivo de log estiver corrompido ou inacessível, não será possível realizar o Tail Log Backup
- Em caso de perda total do storage (disco), não há como recuperar essa porção final

---

## 9 - CONTINUE_AFTER_ERROR

A opção CONTINUE_AFTER_ERROR permite que o SQL Server continue o processo de backup mesmo quando encontra erros de leitura em páginas de dados

### Objetivo

Evitar a falha completa do backup em cenários onde existem problemas pontuais de leitura no banco de dados, permitindo salvar o máximo possível de informações

### Funcionamento

Durante a execução do backup, se o SQL Server encontrar páginas corrompidas ou inacessíveis:

- Sem CONTINUE_AFTER_ERROR:
  - O backup é interrompido e falha

- Com CONTINUE_AFTER_ERROR:
  - O backup continua
  - As páginas com erro são ignoradas
  - Os erros são registrados durante o processo

### Exemplo de uso

`BACKUP DATABASE MinhaBase TO DISK = 'backup.bak' WITH CONTINUE_AFTER_ERROR;`

### Quando utilizar

- Cenários de emergência
- Banco com suspeita de corrupção
- Necessidade de preservar o máximo possível de dados antes de um restore

### Riscos

- O backup pode conter dados corrompidos
- Não garante integridade total do banco
- Pode haver inconsistência nos dados restaurados

### Observações importantes

- Deve ser utilizado apenas como **último recurso**
- Não substitui backups regulares
- Após sua utilização, é recomendado:
  - Executar DBCC CHECKDB
  - Avaliar a extensão da corrupção
  - Planejar a recuperação adequada

### Relação com Tail Log Backup

Em cenários de falha crítica, o CONTINUE_AFTER_ERROR pode ser utilizado em conjunto com o Tail Log Backup para maximizar a recuperação de dados

---

## 10 - Ordem de execução em cenários de falha

Em cenários de corrupção ou falha crítica, a ordem de execução dos backups é fundamental para maximizar a recuperação de dados

### Estratégia recomendada

1. Executar Tail Log Backup

   `BACKUP LOG MinhaBase TO DISK = 'log_tail.trn' WITH NO_TRUNCATE;`

   - Preserva as últimas transações ainda não protegidas
   - Deve ser a primeira ação sempre que o log estiver acessível

2. Executar backup FULL com CONTINUE_AFTER_ERROR

   `BACKUP DATABASE MinhaBase TO DISK = 'backup.bak' WITH CONTINUE_AFTER_ERROR;`

   - Permite concluir o backup mesmo com páginas corrompidas
   - Captura o máximo possível de dados disponíveis

### Justificativa técnica

O backup FULL pode executar um CHECKPOINT implícito, gravando páginas em disco
Em um cenário de corrupção, isso pode:
- Persistir dados inconsistentes
- Alterar o estado do banco
- Dificultar a análise posterior

Por esse motivo, o transaction log deve ser preservado primeiro, pois contém as transações mais recentes

### Cenário alternativo

Caso o transaction log esteja inacessível:

- Não será possível executar o Tail Log Backup
- Nesse caso, deve-se tentar diretamente:

  `BACKUP DATABASE MinhaBase TO DISK = 'backup.bak' WITH CONTINUE_AFTER_ERROR;`

### Resultado esperado

- O Tail Log Backup preserva as transações recentes
- O backup FULL captura os dados disponíveis
- A combinação permite recuperar o máximo possível de informações

### Observações importantes

- Essa abordagem deve ser utilizada apenas em cenários de emergência
- Não garante recuperação completa ou íntegra
- Deve ser seguida de análise com DBCC CHECKDB

### Resumo

Em cenários de falha, o transaction log deve ser priorizado  
A ordem correta de execução é essencial para minimizar perda de dados e aumentar a chance de recuperação

---

## Referências

- [BACKUP (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/backup-transact-sql?view=sql-server-ver16)
- [Visão geral de backup (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-overview-sql-server?view=sql-server-ver16)
- [Backups do log de cauda (tail-log)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/tail-log-backups-sql-server?view=sql-server-ver16)
- [Backups copy-only](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/copy-only-backups-sql-server?view=sql-server-ver16)
- [Especificar se o backup ou restore continua ou para após um erro](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/specify-if-backup-or-restore-continues-or-stops-after-error?view=sql-server-ver16)
