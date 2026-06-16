# A0004 – SQL Server Connectivity Troubleshooting

> **Author:** Rafael Binda  
> **Created:** 2026-02-10  
> **Version:** 5.0  

---

## Descrição

Troubleshooting de conectividade SQL Server.  
Este documento descreve o passo a passo para diagnóstico de conectividade com o SQL Server.

---

## Observações

- Este documento contém informações complementares ao documento [Network Commands](../notes/A0005-network-commands.md)  
- Para informações sobre a versão do SQL Server, verificar o documento [SQL Server Version](../notes/A0006-sql-server-version.md)  
- Para identificar a versão do SQL Server, utilizar o script disponível em [SQL Version and Compatibility](../../../dba-scripts/SQL-instance-information/INST-Q0003-version-and-compatibility.sql)  
- Para identificar se há alguma conexão ativa através de Named Pipe, usar a query disponível em [SQL Active Connections](../../../dba-scripts/SQL-connections/CONN-Q0001-active-connections.sql)  
- Neste capítulo é utilizado o executável disponível em [A0003_X_PortQry.zip](../tools/A0003_X_PortQry.zip)  
  Na página do GitHub, clique em **View** ou **Raw** para baixar o arquivo  
  Após o download, extraia o arquivo PortQry.exe em algum diretório de sua preferência  
  Não há necessidade de realizar instalação  

---

**Este documento apresenta apenas recomendações, não uma lista completa de procedimentos**

---

## 1 - Testar SQL Server Browser (UDP 1434)

→ O SQL Server Browser deve estar ligado  
→ O SQL Server Browser é responsável por informar a porta TCP utilizada pela instância quando ela usa porta dinâmica

### Objetivo

- Verificar se o serviço SQL Server Browser está ativo  
- Confirmar se a porta UDP 1434 está acessível  
- Confirmar se a porta TCP 445 está acessível  
- Descobrir em qual porta está sendo executado o SQL Server  

### Comando

→ Comando digitado utilizando CMD:

```cmd
C:\TESTES\A0003_PortQry>PortQry.exe -n SRVSQLSERVER -p UDP -e 1434
```

### Resultado esperado

```cmd
Querying target system called:
SRVSQLSERVER
Attempting to resolve name to IP address...
Name resolved to 192.168.0.113
querying...
UDP port 1434 (ms-sql-m service): LISTENING or FILTERED
Sending SQL Server query to UDP port 1434...
Server's response:
ServerName SRVSQLSERVER
InstanceName MSSQLSERVER
IsClustered No
Version 16.0.1000.6
tcp 1433
==== End of SQL Server query response ====
```

### Possíveis resultados

- LISTENING  
→ Serviço ativo e respondendo

- NOT LISTENING  
→ Serviço parado ou não instalado

- FILTERED  
→ Firewall bloqueando a porta UDP 1434

---

## 2 - Named Pipe

→ É um "túnel interno" do Windows usado para enviar comandos SQL entre cliente e servidor  
→ É apenas um método de conexão ao SQL Server, alternativo ao TCP  
→ É um protocolo de comunicação do Windows que permite que aplicações conversem entre si através de um "canal" lógico criado na memória do sistema  
→ É uma forma alternativa ao TCP/IP para clientes se conectarem ao SQL  
→ Usa a infraestrutura do Windows (SMB — porta 445)  
→ O caminho típico é algo como: `\\Servidor\pipe\MSSQL$Instancia\sql\query`  
→ `\\Servidor\pipe` não é um compartilhamento de rede comum como:

- `\\Servidor\C$`  
- `\\Servidor\Compartilhado`  
- Isso é um namespace especial do Windows, não é uma pasta compartilhada

→ Ele representa:

- Named Pipes do sistema  
- Criadas dinamicamente em memória  
- Não aparecem em "Gerenciamento de Compartilhamentos"

→ Comando digitado utilizando CMD em um ambiente com Named Pipe ativo:

```cmd
C:\TESTES\PortQry.exe -n Servidor -p TCP -e 445
```

### Resultado

```cmd
Server's response:
ServerName XXXXXX
InstanceName XXXXX
IsClustered No
Version 14.0.1000.169
tcp 59943
np \\XXXXXX\pipe\MSSQL$XXXXX\sql\query
```

### Como funciona internamente

→ Quando o serviço da instância XYZZZ do Microsoft SQL Server inicia ele cria dinamicamente:

```cmd
\\.\pipe\MSSQL$Instancia\sql\query
```

→ Existem comandos que podem ser executados no PowerShell ou usando Sysinternals que listam as Pipes existentes

### Interpretação de segurança

→ Se 445 estiver fechada:

- Named Pipes não é acessível remotamente  
- Menor superfície de ataque

→ Se 445 estiver aberta:

- SMB está exposto  
- Named Pipes pode ser usado remotamente

### Named Pipe listado no EventViewer mesmo estando desabilitado

→ 1º - No SQL Server Configuration Manager → Protocols for MSSQLSERVER → Named Pipes = Disabled  
→ 2º - Ao iniciar o Windows no Event Viewer é possível ver a seguinte mensagem:  
`Server local connection provider is ready to accept connection on [ \\.\pipe\SQLLocal\MSSQLSERVER ]`

→ Porque ele está sendo listado se está como Disabled?

- Isso é um Named Pipe interno/local, usado pelo próprio mecanismo do Microsoft SQL Server para conexões locais internas  
- Ele faz parte do Shared Memory / Local DB stack, e não do Named Pipes configurável que você vê no  
  `SQL Server Configuration Manager → Protocols for MSSQLSERVER`  
- Ele continua funcionando mesmo com Named Pipes desabilitado

---

## 3 - Identificar a porta TCP real do SQL Server

→ Para descobrir a porta real:

### Opção 1 – SQL Server Configuration Manager

→ SQL Server Network Configuration → Protocols for MSSQLSERVER → TCP/IP → Properties → IPAll  
→ Verificar:

- TCP Dynamic Ports  
- TCP Port  

→ Se o TCP 1433 não estiver funcionando, pode ser que a instância esteja usando porta dinâmica

### Opção 2 – Executando dentro do SQL Server

→ Deve estar conectado remotamente — não pode estar logado no SSMS dentro do próprio servidor de banco

```sql
SELECT local_net_address, local_tcp_port
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

### Resultado esperado

```cmd
local_tcp_port
49732
```

---

## 4 - Testar a porta real usando o PortQry

→ Exemplo (supondo porta 1433):

→ Comando digitado utilizando CMD:

```cmd
C:\TESTES\A0003_PortQry>PortQry.exe -n SRVSQLSERVER -p TCP -e 1433
```

### Resultado esperado

```cmd
Querying target system called:
SRVSQLSERVER
Attempting to resolve name to IP address...
Name resolved to 192.168.0.113
querying...
TCP port 1433 (ms-sql-s service): LISTENING
```

### Possíveis resultados

- LISTENING  
→ Serviço ativo e respondendo

- NOT LISTENING  
→ Serviço parado ou não instalado

- FILTERED  
→ Firewall bloqueando a porta TCP testada

---

## 5 - Testar intervalo de portas

→ Objetivo:

- Testar múltiplas portas simultaneamente  
- Útil quando não se sabe a porta exata

→ Comando digitado utilizando CMD:

```cmd
C:\TESTES\A0003_PortQry>PortQry.exe -n SRVSQLSERVER -p TCP -r 1400:1500
```

### Resultado esperado

```cmd
Querying target system called:
SRVSQLSERVER
Attempting to resolve name to IP address...
Name resolved to 192.168.0.113
querying...
TCP port 1430 (unknown service): NOT LISTENING
TCP port 1431 (unknown service): NOT LISTENING
TCP port 1432 (unknown service): NOT LISTENING
TCP port 1433 (ms-sql-s service): LISTENING
TCP port 1434 (ms-sql-m service): NOT LISTENING
TCP port 1435 (unknown service): NOT LISTENING
```

---

→ **CUIDADO COM CTRL+C / CTRL+V**  
→ Ao copiar comandos, o hífen (-) pode ser substituído automaticamente por outro caractere semelhante (–), causando erro no comando  
→ Sempre verifique se o hífen utilizado é o padrão do teclado

---

## Referências

- [Erro de rede ou instância específica ao conectar ao SQL Server](https://learn.microsoft.com/pt-br/troubleshoot/sql/database-engine/connect/network-related-or-instance-specific-error-occurred-while-establishing-connection)
- [Usando a ferramenta PortQryUI com SQL Server](https://learn.microsoft.com/pt-br/troubleshoot/sql/database-engine/connect/using-portqrytool-sqlserver)
- [Ferramenta PortQry – linha de comando (versão 2.0)](https://learn.microsoft.com/pt-br/troubleshoot/windows-server/networking/portqry-command-line-port-scanner-v2)
- [Serviço SQL Server Browser](https://learn.microsoft.com/pt-br/sql/tools/configuration-manager/sql-server-browser-service?view=sql-server-ver16)
