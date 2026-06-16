# A0003 – SQL Server Installation

> **Author:** Rafael Binda  
> **Created:** 2026-02-10  
> **Version:** 3.0  

---

## Descrição

Documento com informações para instalação do SQL Server seguindo boas práticas.

---

## Observações

- Necessário fazer o download do SQL Server no site da Microsoft.
- Necessário fazer o download do SQL Server Management Studio no site da Microsoft.  
- Para informações a respeito de Collation, verificar o arquivo [SQL Server Collation](../notes/A0002-collation.md)

---

## Meu cenário

→ Versão: SQL Server 2022  
→ Atualização: Cumulative Update 23 (CU23)  
→ Data de liberação: Janeiro 22, 2026  
→ Edição: Enterprise Developer  
→ Sistema Operacional: Windows Server 2025 Evaluation  
→ Ambiente virtualizado com Hyper-V  

---

## 1. Servidor Windows Server 2025 Evaluation

→ Servidor deve estar ingressado em Domínio (AD)  
→ Evitar ambiente WORKGROUP em produção  
→ Criar conta dedicada no AD para o serviço SQL  
→ Conta não deve ser Administrator  

Durante a instalação:  
O usuário que está instalando o SQL Server precisa de privilégios de administrador.

---

### IMPORTANTE

Durante a configuração dos serviços, deve especificar uma conta de usuário (de preferência de domínio) que não seja administradora.

---

### Por que não usar Administrador?

**Risco de Segurança:**  
Se o serviço do SQL Server for comprometido (por exemplo, via injeção de SQL), o atacante terá controle total da máquina ou do domínio.

**Melhores Práticas:**  
A recomendação é usar uma conta de usuário de domínio comum ou, preferencialmente, Managed Service Accounts (MSAs) ou Virtual Accounts.

---

→ Configurar senha para NUNCA EXPIRAR  
→ Remover privilégios desnecessários  
→ Servidor preferencialmente dedicado ao SQL Server  

---

## 2 - Ao executar o instalador do SQL Server – Tipo de Instância

→ Definir se será utilizada Instância Padrão ou Nomeada  
→ Instância Padrão utiliza porta 1433  
→ Instância Nomeada deve obrigatoriamente utilizar porta fixa  
→ Não utilizar portas dinâmicas em ambiente de produção  
→ Configurar porta fixa no SQL Server Configuration Manager  
→ Validar liberação da porta no Firewall do Windows  
→ Validar liberação da porta no Firewall de rede (se existir)

---

## 3 - Ao executar o instalador do SQL Server - (Feature Selection)

→ Em Instance Features marcar os itens:

- [X] Database Engine Services  
    → Servidor de banco de dados relacional  
    → Serviço principal do SQL Server  

-    [x] SQL Server Replication  
        → Replicação para banco de dados distribuído  
        → Permite copiar e sincronizar dados entre servidores  

 -   [x] AI Services and Language Extensions  
        → Disponível só no SQL 2025  
        → Permite rodar scripts externos dentro do SQL Server (Python, R, Java)  

 -   [x] Full-Text and Semantic Extractions for Search  
        → Permite buscas avançadas em texto  
        → Sem isso, o LIKE '%texto%' fica limitado e lento  

 -   [x] PolyBase Query Service for External Data  
        → Permite consultar dados externos como se fossem tabelas  
        → Arquivos CSV, Hadoop, Azure Blob  
        → Muito usado para integração e Data Lake  

- [ ] Analysis Services  
    → Usado para BI avançado (Power BI, relatórios analíticos)  
    → Se não usar, não instalar  

---

### Shared Features

- [X]  Integration Services  
    → Ferramenta de ETL  
    → Eu uso o Pentaho, mas instalar para possível estudo no futuro  

-    [ ] Scale Out Master  
        → Controlador central do SSIS Scale Out  
        → A princípio não instalar  
        → Só precisa se for montar ambiente distribuído de ETL pesado  

-    [ ] Scale Out Worker  
        → Máquina que executa pacotes SSIS distribuídos  
        → Só faz sentido em arquitetura grande com processamento distribuído  

---

### Diretórios

Instance root directory  
→ Pasta base onde os arquivos da instância serão instalados  
→ Isso não é onde ficam os bancos (.mdf/.ldf) por padrão  

Shared feature directory  
→ Pasta onde ficam componentes compartilhados entre instâncias  
→ SSMS components  
→ Client tools  
→ DLLs compartilhadas  
→ Integration Services  

Shared feature directory (x86)  
→ Versão 32 bits dos componentes compartilhados  
→ Quase ninguém usa x86  
→ Pode deixar padrão  
→ Só é usado para compatibilidade antiga  

---

## 4 - Server Configuration – Serviços

→ SQL Server Agent  
    → Usar a conta que foi criada no Windows (.\USRSQLSERVER)  

→ SQL Server Database Engine  
    → Usar a conta que foi criada no Windows (.\USRSQLSERVER)  

→ As demais pode deixar como está, sem entrada de um usuário específico  

→ Startup Type como Automatic para:  
    → SQL Server Database Engine  
    → SQL Server Agent  

---

→ Selecionar:  
- [X] Grant Perform Volume Maintenance Tasks privilege to SQL Server Database Engine Service  

→ Concede ao serviço do SQL Server a permissão para usar Instant File Initialization (IFI)  
→ Melhora performance em:
- Crescimento automático (autogrowth)
- Restore
- Criação de banco grande

---

## 5 - Collation

→ Definir Collation antes da instalação  
→ Padrão recomendado moderno: `Latin1_General_100_CI_AI_SC_UTF8`  
→ Não alterar Collation apos ambiente estar em produção  
→ Validar compatibilidade com aplicações antes da definição  
→ Para informações a respeito de collation verificar o arquivo `notes\A0002_collation.md`

---

## 6 - Database Engine Configuration - Server Configuration

→ Escolher Mixed Model (SQL Server and Windows authentication)  
→ Nesse momento é definida a senha padrão do usuário "sa"  
- Dica: Colocar a mesma senha da conta do serviço (.\USRSQLSERVER)
    
→ Adicionar o usuário corrente  
→ Adicionar o usuário da conta do serviço (.\USRSQLSERVER)  
→ Todos os usuários que forem adicionados aqui terão permissão de sysadmin  
→ Ambiente de produção tem que ter alguns cuidados a mais mas é assunto para o futuro

---

## 7 - Database Engine Configuration - Data Directories

→ Data root não alterar  
→ System database não alterar  
→ Os demais definir um diretório que fique fácil para uso no dia a dia e se possível:
- Dados em um disco exclusivo
- Log em um disco exclusivo
- Backup em um disco exclusivo

---

## 8 - Database Engine Configuration - TempDB

→ Para estudo nesse momento não alterar nada  
→ Número de arquivos = número de processadores lógicos (até 8)  
    → Se contention persistir após 8 arquivos: aumentar em múltiplos de 4  
→ Todos os arquivos devem ter o mesmo tamanho inicial e o mesmo autogrowth  
    → O algoritmo `proportional fill` depende de arquivos iguais para distribuir alocações de forma equilibrada  
→ Tamanho inicial: pré-alocar conforme o workload típico do ambiente  
    → Valor padrão do setup: 8 MB — ajustar para evitar autogrowth frequente durante execução  
→ Autogrowth fixo recomendado pela Microsoft: **64 MB** por arquivo  
    → Ambientes maiores podem usar 256 MB ou 512 MB — sempre fixo, nunca percentual  
→ SQL Server 2016 e posteriores: trace flags 1118 e 1117 **não são mais necessários**  
    → Alocação uniforme e crescimento simultâneo dos arquivos já são built-in nessa versão  

---

## 9 - Database Engine Configuration - MaxDOP
    
→ Seguir recomendacao Microsoft:  
- Até 8 cores: MaxDOP = numero de cores  
- Acima de 8 cores: MaxDOP = 8  
- Sempre validar se ha multiplos NUMA nodes
  
→ Cuidado com o MaxDOP (verificar link disponível em Referências)

---

## 10 - Database Engine Configuration - Memory

→ Marcar (*) Recommended  
→ Definir memória máxima manualmente  
→ Reservar entre 4GB e 8GB para o Sistema Operacional  
- Exemplo: servidor 32GB RAM
→ Max Server Memory: 24576 MB (24GB)

---

## 11 - Database Engine Configuration - FILESTREAM
	
→ Permite armazenar arquivos grandes (BLOBs) no sistema de arquivos NTFS, mas controlados pelo SQL Server.  
→ Em vez de guardar tudo dentro da tabela como VARBINARY(MAX), o SQL salva o arquivo fisicamente no disco e mantém o controle transacional.  
→ Usado quando você precisa armazenar:  
- PDFs  
- Imagens  
- Vídeos  
- Documentos grandes  

- [x] Enable FILESTREAM for Transact-SQL access  
→ Permite acessar via T-SQL.

- [x] Enable FILESTREAM for file I/O streaming access  
→ Permite acesso direto via Windows API.

- [ ] Allow remote clients  
→ Permite acesso remoto aos arquivos.   
→ Gera problemas sérios de segurança

---

## 12 - Checar o ambiente
	
→ 1 - Inciar os serviços  
→ 2 - Abrir o CMD e digitar o comando `sqlcmd -?` 
- Resultado: Vai listar todas as opções disponíveis para uso

→ 3 - Abrir o CMD e digitar o comando `sqlcmd`
- Resultado: Vai conectar no servidor de banco de dados (se estiver mixed mode

---

## 13 - Instalação de uma segunda/terceira instância para testes com Collation

- Criar um novo usuário denominado `SQLServiceCollation` e definir que a senha vai expirar nunca (usado para testes com uma instância com Collation diferente)

- **Instalação:**  
  → Installation → New Stand Alone  
  → Installation Type → Perform a new installation  
  → Edition → Developer  
  → Licence term → Accept  
  → Azure Extension → Marcação do Azure é só se for para que ocorra cobrança na conta do Azure  

- **Feature Selection:**  
  → Marcar no mínimo `Database Engine Services`, `Full-Text`  
  → Alterar `Instance root` para: `C:\Microsoft SQL Server Instance III`  

- **Instance and Configuration:**  
  → Nomeia a instância: `MSSQLSERVERIII` (3ª Instância)  

- **Server Configuration:**  
  → Alterar `Account Name` para usar o usuário `SQLServiceCollation` e informar a senha XYZ  
  → Startup Type: Automatic  
    → SQL Server Agent  
    → SQL Server Database Engine  
  → Marcar `[X] Grant Perform Volume Maintenance Tasks privilege to SQL Server Database Engine Service`  
    → Este privilégio habilita **Instant File Initialization** evitando a inicialização com zeros das páginas de dados.  
    → Pode levar a divulgação de informações antigas deletadas.  
  → Definir Collation: `Latin1_General_100_CI_AI_SC_UTF8`  

- **Database Engine Configuration:**  
  → Server Configuration  
    → Marcar Mixed Mode + informar senha  
    → Add Current User  
    → Add User `SQLServiceCollation`  

  → Data Directories  
    → Root: `C:\Microsoft SQL Server Instance III`  
    → User database directory: `C:\Microsoft SQL Server Instance III\MSSQL_Data`  
    → User database log directory: `C:\Microsoft SQL Server Instance III\MSSQL_Log`  
    → Backup directory: `C:\Microsoft SQL Server Instance III\MSSQL_Backup`  

  → TempDB  
    → Number of files: igual ao número de processadores  
    → Initial size: 100 MB (ajustar conforme necessário)  
    → Autogrowth: 100 MB  
    → Data directories: `C:\Microsoft SQL Server Instance III\MSSQL_Temp`  
    → TempDB Log File Initial size: 100 MB  
    → TempDB Autogrowth: 100 MB  
    → Log directory: `C:\Microsoft SQL Server Instance III\MSSQL_Temp_Log`  
    → Atenção ao MaxDOP  

  → Memory  
    → Alterar para Recommended  
    → Marcar `Click here to accept the recommended memory configurations for the SQL Server Database Engine`  

  → FILESTREAM  
    → Habilitar as duas primeiras opções

---

## 14 - Instalar o SSMS - SQL Server Management Studio

---

## Referências

- [Edições e componentes do SQL Server 2022](https://learn.microsoft.com/pt-br/sql/sql-server/editions-and-components-of-sql-server-2022?view=sql-server-ver16)
- [Configurar o grau máximo de paralelismo (MaxDOP)](https://learn.microsoft.com/pt-br/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option?view=sql-server-ver16)
- [Banco de dados tempdb – configuração e otimização](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/tempdb-database?view=sql-server-ver16)
