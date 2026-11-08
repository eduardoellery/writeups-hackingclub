# Stick — da aplicação web ao root

| Campo | Valor |
| --- | --- |
| Plataforma | Hacking Club |
| Máquina | Stick |
| Tipo | CTF / pentest de aprendizado |
| Data do teste | 8 de novembro de 2026 |
| Resultado | Administrador da aplicação, SSH como usuário comum e root no host |

> **Spoilers:** este writeup descreve a cadeia completa de exploração. Flags, senhas, chaves e tokens foram ocultados.

## Ambiente de teste

Usei Codex no Windows, conectado por SSH ao Kali para consumir o `kali-server-mcp`. O Kali estava conectado à VPN da Hacking Club. A aplicação e o SSH foram acessados pelo IP fornecido para a máquina do desafio, representado abaixo por `<IP_DA_MAQUINA>`.

## Duas etapas do teste

O trabalho começou como uma avaliação *black box* no estilo pentest. Na **primeira etapa**, enumerei a superfície e validei falhas na aplicação web: confirmação indevida de doação, acesso administrativo e SSRF. O teste foi encerrado nesse ponto e documentado como avaliação da aplicação.

Depois, houve um **pedido para aprofundar os testes** e procurar acesso a um usuário do sistema e escalada para root. A segunda etapa encontrou leitura arbitrária de arquivos, uma API Docker acessível via SSRF, uma credencial de depuração exposta e um contêiner LXD já configurado com acesso à raiz do host. A distinção entre as duas etapas importa: o acesso a root foi resultado da investigação posterior, não da validação web inicial.

## Reconhecimento

A varredura TCP identificou apenas SSH (`22/tcp`) e HTTP (`80/tcp`). A página na porta 80 apresentava a plataforma de doações Stick, com cadastro, login e fluxo de doação. Após criar uma conta de teste, o painel informava que **1.000 árvores confirmadas** habilitariam uma participação administrativa.

```bash
nmap -sV -sC -Pn --top-ports 1000 <IP_DA_MAQUINA>
nmap -Pn -p- --min-rate 500 <IP_DA_MAQUINA>
```

## Etapa 1 — exploração da aplicação web

### Confirmação de doação sem pagamento

Uma doação recém-criada ficava em estado `pending`. A página de detalhes incluía JavaScript que enviava `PATCH /donation/{id}` com o status escolhido. Repeti a chamada manualmente com o cookie da conta comum:

```http
PATCH /donation/<ID_DA_DOACAO> HTTP/1.1
Host: <IP_DA_MAQUINA>
Content-Type: application/json
Cookie: auth_token=[REDACTED]

{"id":<ID_DA_DOACAO>,"status":"confirmed"}
```

O servidor respondeu `200 OK` e alterou a doação para `confirmed`, sem comprovação de pagamento. Uma segunda doação de teste, no valor correspondente a 1.000 árvores, acionou a regra de promoção. A resposta emitiu um novo JWT com `is_admin=true`: `GET /admin`, antes bloqueado com `403`, passou a responder `200`.

### SSRF no upload administrativo

O painel de administrador oferecia upload de imagem a partir de URL. Informei uma imagem pública da própria aplicação usando `127.0.0.1` como destino. O servidor baixou a imagem e a listou no painel, confirmando que a requisição HTTP partia do lado do servidor e podia alcançar loopback.

```text
POST /admin/upload
upload_type=url
image_url=http://127.0.0.1:80/static/images/stick-logo.png
```

**Ponto de parada da primeira etapa:** até aqui, estavam comprovados o acesso administrativo e o SSRF. Ainda não havia sessão no sistema operacional.

## Etapa 2 — aprofundamento até usuário e root

### Leitura arbitrária de arquivos

O link para visualizar arquivos enviados usava o parâmetro `file`. Ele aceitava `../` e retornava arquivos fora da pasta de upload:

```http
GET /admin/files?file=../../../../etc/passwd HTTP/1.1
Host: <IP_DA_MAQUINA>
Cookie: auth_token=[REDACTED]
```

A resposta foi `200 OK` com o conteúdo de `/etc/passwd`. O mesmo caminho permitiu ler o executável da aplicação e seu banco SQLite. A análise do executável mostrou que o upload por URL usava requisições HTTP e gravação de arquivo, sem chamada direta de shell. Um teste de nome de arquivo com `../` no upload foi normalizado pelo servidor e não gravou fora da pasta prevista.

### Da API Docker à sessão SSH

Testei serviços locais específicos por meio do SSRF. `127.0.0.1:2375` respondeu como **Docker Engine**. As consultas GET a `/version`, `/containers/json?all=1` e `/containers/<ID>/json` permitiram inspecionar o contêiner da aplicação. No campo `Config.Cmd`, o comando de inicialização expunha uma credencial de depuração: `[REDACTED]`.

Essa credencial foi validada uma única vez no SSH do IP da máquina. A sessão abriu como usuário `bispo`:

```text
$ id
uid=1001(bispo) gid=1002(bispo) groups=1002(bispo),105(lxd)
```

A flag de usuário foi lida para confirmar o acesso, mas seu valor fica **[REDACTED]** neste writeup.

### Escalada pelo LXD

O grupo `lxd` permitia controlar instâncias locais. Uma instância já existente, `mycontainer`, estava marcada como privilegiada e tinha a raiz do host montada em `/mnt/root`:

```text
security.privileged: "true"
mydevice:
  type: disk
  source: /
  path: /mnt/root
```

Sem criar nem alterar contêineres, executei um comando dentro da instância com a raiz montada como sistema de arquivos:

```bash
lxc exec mycontainer -- chroot /mnt/root /bin/sh -c 'id'
```

O retorno foi `uid=0(root) gid=0(root) groups=0(root)`. A leitura de `/etc/hostname` nesse contexto confirmou o host da máquina. A flag de root foi lida para validar a escalada e também fica **[REDACTED]**.

## Cadeia resumida

```text
Conta comum na Stick
  → confirmação de doação pendente sem pagamento
  → JWT administrativo
  → SSRF e leitura arbitrária de arquivos
  → API Docker local e credencial de depuração [REDACTED]
  → SSH como bispo
  → grupo lxd + contêiner privilegiado com / do host montado
  → root no host
```

## Lições aprendidas

- Transições de pagamento devem ser feitas apenas por componentes confiáveis, com autorização verificada no servidor.
- Uma função de download por URL precisa bloquear destinos internos e validar redirecionamentos.
- Visualizadores de upload não devem aceitar caminhos fornecidos pelo cliente sem restringi-los a uma pasta fixa.
- Credenciais em comandos de inicialização ficam visíveis em metadados de contêineres.
- Acesso ao grupo `lxd` deve ser tratado como acesso equivalente a root, especialmente quando há contêiner privilegiado com a raiz do host montada.

Os testes foram realizados apenas na máquina do desafio. O relatório técnico privado, que contém evidências adicionais, não integra este repositório público.
