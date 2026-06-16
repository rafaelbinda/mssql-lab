# A0005 – Network Commands

> **Author:** Rafael Binda  
> **Created:** 2026-02-10  
> **Version:** 2.0  

---

## Descrição

Documento com comandos de diagnóstico de rede para uso no Prompt de Comando (CMD) do Windows. Abrange resolução de nomes e IP, identificação de máquinas na rede, instalação e uso do Telnet para teste de portas, e listagem de conexões ativas.

---

## Observações

- Este documento contém informações complementares ao documento [SQL Server Connectivity Troubleshooting](../notes/A0004-sql-server-connectivity-troubleshooting.md)

---

**Este documento apresenta apenas recomendações, não uma lista completa de procedimentos.**

---

## 1 - Descobrir o IP de um host usando CMD

→ Opção 1 – resolução via ICMP:

```cmd
ping google.com.br
```

### Resultado esperado

```cmd
Pinging google.com.br [172.217.172.163] with 32 bytes of data:
Reply from 172.217.172.163: bytes=32 time=15ms TTL=117
Reply from 172.217.172.163: bytes=32 time=16ms TTL=117
Reply from 172.217.172.163: bytes=32 time=15ms TTL=117
Reply from 172.217.172.163: bytes=32 time=16ms TTL=117
```

→ Opção 2 – resolução via DNS:

```cmd
nslookup google.com
```

### Resultado esperado

```cmd
Server:  mhnet-costumer-dns-a.mhnet.com.br
Address:  187.45.96.96

Non-authoritative answer:
Name:    google.com
Addresses:  2800:3f0:4001:801::200e
          142.250.78.142
```

---

## 2 - Descobrir o nome de uma máquina na rede

→ Comando digitado utilizando CMD:

```cmd
ping -a 192.168.1.109
```

### Resultado esperado

```cmd
Pinging SRVSQLSERVER [192.168.0.109] with 32 bytes of data:
Reply from 192.168.0.109: bytes=32 time<1ms TTL=128
Reply from 192.168.0.109: bytes=32 time<1ms TTL=128
Reply from 192.168.0.109: bytes=32 time<1ms TTL=128
Reply from 192.168.0.109: bytes=32 time<1ms TTL=128
```

---

## 3 - Habilitar Telnet pelo Prompt de Comando

→ Abra o Prompt de Comando (CMD) como Administrador e execute o seguinte comando e aguarde a instalação:

```cmd
dism /online /Enable-Feature /FeatureName:TelnetClient
```

### Resultado esperado (Telnet não instalado)

`'telnet' não é reconhecido como um comando interno ou externo, um programa operável ou um arquivo em lotes.`

### Resultado esperado (instalação bem-sucedida)

```cmd
Deployment Image Servicing and Management tool
Version: 10.0.26100.5074

Image Version: 10.0.26100.32230

Enabling feature(s)
[==========================100.0%==========================]
The operation completed successfully.
```

### Saber se uma porta está ativa usando Telnet

→ Comando digitado utilizando CMD:

```cmd
telnet 192.168.0.109 1433
```

### Resultado esperado

→ Quando você usa telnet para se conectar a uma porta TCP de um serviço como o SQL Server:

- O telnet abre a conexão na porta 1433
- O SQL Server não "fala" nada automaticamente, então a tela fica em branco/presa, esperando que você envie algum comando
- Isso significa que a conexão TCP foi estabelecida com sucesso, ou seja, a porta está aberta e acessível
- Se você vir mensagens de erro como `Could not open connection`, há algum bloqueio de firewall, rede ou configuração do SQL

---

## 4 - Listar conexões abertas e portas em escuta

→ Comando digitado utilizando CMD:

```cmd
netstat -ano | findstr LISTENING
```

- `-a` → Mostra todas as conexões e portas em escuta (listening)
- `-n` → Exibe os endereços e portas em formato numérico (sem tentar resolver nomes de host ou serviços)
- `-o` → Inclui o PID (Process ID), ou seja, o identificador do processo que está usando aquela conexão/porta

### Resultado esperado

```cmd
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       952
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       7212
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       1196
```

→ Após saber o PID listado no comando anterior, é possível descobrir qual programa está usando a porta:

```cmd
tasklist /FI "PID eq 7212"
```

### Resultado esperado

```cmd
Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
sqlservr.exe                  7212 Services                   0    769.400 K
```

---

## Referências

- [Comando ping – referência oficial](https://learn.microsoft.com/pt-br/windows-server/administration/windows-commands/ping)
- [Comando nslookup – referência oficial](https://learn.microsoft.com/pt-br/windows-server/administration/windows-commands/nslookup)
- [Comando netstat – referência oficial](https://learn.microsoft.com/pt-br/windows-server/administration/windows-commands/netstat)
- [Cliente Telnet – referência oficial](https://learn.microsoft.com/pt-br/windows-server/administration/windows-commands/telnet)
