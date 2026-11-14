# Ouroboros — DSC Pull Server expõe credencial

| Campo | Valor |
| --- | --- |
| Plataforma | Hacking Club |
| Máquina | Ouroboros |
| Tipo | CTF Windows / Active Directory |
| Data do teste | 12 e 14 de novembro de 2026 |
| Resultado | Acesso como `svc_mgmt` e flag de usuário; escalada a `SYSTEM` não comprovada |

> **Spoilers:** este texto descreve somente etapas validadas. Flags e senhas foram ocultadas.

## Ambiente de teste

O Kali estava conectado à VPN da Hacking Club. O IP temporário da máquina aparece abaixo como `<IP_DA_MAQUINA>`.

## Reconhecimento

Uma varredura das mil portas TCP mais comuns encontrou DNS (`53`), Kerberos (`88`), LDAP (`389`), SMB (`445`), RDP (`3389`) e IIS (`8080`), entre outros serviços de Active Directory. O LDAP identificou o domínio `ouroboros.hc` e o host `MGMT01`.

```bash
nmap -Pn -sT -sV --top-ports 1000 <IP_DA_MAQUINA>
dig @<IP_DA_MAQUINA> ouroboros.hc AXFR
```

A transferência de zona DNS respondeu com 22 registros. Um TXT em `_dsc._tcp.ouroboros.hc` indicava `http://MGMT01.ouroboros.hc:8080/PSDSCPullServer.svc`.

## Exploração comprovada

O serviço DSC respondeu sem autenticação. A coleção `Configurations` listou um identificador de configuração e o caminho de um arquivo MOF:

```bash
curl http://<IP_DA_MAQUINA>:8080/PSDSCPullServer.svc/Configurations
curl http://<IP_DA_MAQUINA>:8080/Configuration/<ID_DA_CONFIGURACAO>.mof
```

O MOF continha uma instância `MSFT_Credential` com `UserName = "OUROBOROS\\svc_mgmt"` e `Password = "[REDACTED]"`. A credencial foi validada por SMB e LDAP. A conta pertencia a `Remote Desktop Users`, e a sessão RDP abriu no host `MGMT01`.

```bash
nxc smb <IP_DA_MAQUINA> -d ouroboros.hc -u svc_mgmt -p '[REDACTED]'
xfreerdp /v:<IP_DA_MAQUINA> /u:svc_mgmt /d:ouroboros.hc /p:'[REDACTED]' /cert:ignore
```

Na sessão, `C:\Users\svc_mgmt\Desktop\user.txt` continha a flag de usuário **[REDACTED]**.

## Limite da validação de privilégio

A descrição da máquina apontava para abuso de `DnsAdmins` com `ServerLevelPluginDll`. Nesta instância, a consulta LDAP mostrou o grupo `DnsAdmins` **sem membros** e `whoami /groups` não incluiu esse grupo no token de `svc_mgmt`. A consulta `dnscmd . /info /serverlevelplugindll` retornou `ERROR_ACCESS_DENIED (5)`. A ACL do grupo também não mostrou permissão de escrita concedida a `svc_mgmt`. Portanto, não foi validado carregamento de DLL, execução como `SYSTEM` nem leitura da flag de root.

### Retomada em 14 de novembro

Com um novo IP temporário, repeti o AXFR e a consulta ao DSC Pull Server: a zona, a configuração e a credencial eram as mesmas. Uma coleta de ACLs do Active Directory não encontrou direitos de `svc_mgmt` sobre outros objetos. A sessão RDP confirmou novamente que o token contém apenas grupos de usuário comum, incluindo `Remote Desktop Users`; `DnsAdmins` continuava sem membros e `dnscmd` retornou `ERROR_ACCESS_DENIED`.

Examinei os serviços, tarefas agendadas e arquivos da aplicação DSC. A aplicação em `C:\inetpub\PSDSCPullServer` servia o MOF por código em `Global.asax`; a conta atual não possuía escrita nessa pasta. As verificações locais não indicaram um serviço ou tarefa personalizados graváveis. Uma tentativa única de reutilizar a credencial em `Administrator` falhou.

O serviço de impressão aceitou uma coerção PrinterBug e o host autenticou-se de volta como `MGMT01$` no listener de teste. O relay dessa autenticação para LDAP falhou: a sessão SMB solicitou assinatura. Uma segunda tentativa com remoção de MIC também falhou. Nenhuma permissão ou associação de grupo foi alterada. As verificações de PrintNightmare e NoPAC não indicaram vulnerabilidade explorável. A leitura direta de `C:\Users\Administrator\Desktop\root.txt` retornou `Access is denied`.

**Estado após a retomada:** a flag de usuário continua comprovada; a chave de administrador e a flag de root não foram obtidas.

## Cadeia resumida

```text
AXFR público
  → localização do DSC Pull Server
  → coleção Configurations sem autenticação
  → MOF com credencial em texto claro [REDACTED]
  → RDP como svc_mgmt
  → flag de usuário [REDACTED]
```

## Lições aprendidas

- Restringir transferências de zona DNS a servidores secundários autorizados.
- Proteger coleções e arquivos do DSC Pull Server com autenticação e controle de acesso.
- Evitar credenciais em texto claro em arquivos MOF distribuídos a clientes.
- Confirmar privilégios efetivos antes de concluir uma cadeia de escalada descrita em pistas.
