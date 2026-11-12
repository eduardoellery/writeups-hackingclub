# Crux — BOLA, Handlebars e php-fpm como root

| Campo | Valor |
| --- | --- |
| Plataforma | Hacking Club |
| Máquina | Crux |
| Tipo | CTF |
| Data do teste | 12 de novembro de 2026 |
| Resultado | Conta owner, execução como node e root no host |

> **Spoilers:** este writeup descreve a cadeia completa de exploração. Flags, senha de teste e tokens foram ocultados.

## Ambiente de teste

Usei Codex no Windows, conectado por SSH ao Kali para executar os testes. O Kali estava conectado à VPN da Hacking Club. O IP temporário do alvo está representado por `<IP_DA_MAQUINA>`.

## Reconhecimento

A Kali tinha rota para o alvo pela interface `tun0`. O scan completo de TCP mostrou apenas `22/tcp` (OpenSSH) e `80/tcp` (nginx). A resposta de `http://<IP_DA_MAQUINA>/` redirecionava para `http://crux.hc/`, então as requisições seguintes usaram `Host: crux.hc`.

```bash
nmap -Pn -sS -p- --min-rate 1000 -T4 <IP_DA_MAQUINA>
nmap -Pn -sV -p22,80 <IP_DA_MAQUINA>
```

O arquivo `/static/js/api-client.js` revelou os endpoints `/api/v1/register`, `/login`, `/me`, `/leaderboard`, `/pass/renew`, `/config/export` e `/config/restore`. O `robots.txt` indicava `/console`.

## BOLA e tomada de conta

Criei uma conta comum (`member_id=4`). O ranking expôs os IDs 1–3 de outros membros. Uma chamada autenticada a `POST /api/v1/pass/renew` com `{"member_id":1}` retornou `204` e um novo cookie `session_token`. Em seguida, `GET /api/v1/me` identificou a sessão como a do membro 1. O servidor usou o ID fornecido pelo cliente para escolher a conta da nova sessão, sem verificar se correspondia ao usuário autenticado.

Os IDs 1–3 eram membros comuns. A enumeração de IDs válidos encontrou `member_id=593`; `/api/v1/me` confirmou `username=dana.reeves` e `role=owner`. Com essa sessão, `/console` e `/api/v1/config/export` passaram a responder `200`.

**Causa:** autorização no nível do objeto ausente na renovação do passe. **Impacto:** uma conta comum podia obter sessão de outro membro, inclusive a conta `owner`.

## Injeção de AST do Handlebars

O endpoint de exportação retornou um `backup` em Base64. Após decodificação, o conteúdo era uma AST JSON de Handlebars. A quarta entrada de `body` era um `MustacheStatement` com `lookup` e um parâmetro `NumberLiteral` de valor `0`. Troquei esse valor numérico por uma expressão JavaScript e reenviei a AST codificada em Base64 para `POST /api/v1/config/restore`.

Valor de prova usado no `NumberLiteral.value`:

```js
{},{})) + process.getBuiltinModule('child_process').execFileSync('id').toString() //
```

O campo `preview` da resposta trouxe `uid=1001(node)`. O compilador inseriu o valor de `NumberLiteral` no JavaScript gerado sem restringi-lo a número, conforme a [advisory do projeto Handlebars](https://github.com/handlebars-lang/handlebars.js/security/advisories/GHSA-2w6w-674q-4c4q). **Impacto:** execução de comandos no processo Node.js.

Como o endpoint de restauração persiste o template, guardei a exportação original antes do teste e a restaurei ao final.

A flag de usuário foi lida em `/home/node/user.txt` para confirmar o acesso; seu valor fica **[REDACTED]** neste writeup.

## Escalação pelo php-fpm do WordPress interno

Com execução como `node`, `ss -lntp` mostrou nginx em `127.0.0.1:8080` e php-fpm em `127.0.0.1:9001`. O nginx interno serve `/var/www/wordpress`. A configuração `/etc/php/8.5/fpm/pool.d/wp.conf` confirmou:

```ini
[wp]
user = root
group = root
listen = 127.0.0.1:9001
```

O processo `node` alcançava o socket TCP do pool. Criei um arquivo PHP temporário em `/tmp/crux-root.php` e fiz uma requisição FastCGI diretamente ao pool, definindo `PHP_VALUE=auto_prepend_file=/tmp/crux-root.php` e `SCRIPT_FILENAME=/var/www/wordpress/wp-content/index.php`. O arquivo WordPress existente serviu como script principal; o php-fpm executou primeiro o arquivo indicado por `auto_prepend_file`.

```sh
printf '%s' '<?php echo shell_exec("id"); ?>' > /tmp/crux-root.php
SCRIPT_FILENAME=/var/www/wordpress/wp-content/index.php \
SCRIPT_NAME=/wp-content/index.php REQUEST_METHOD=GET \
PHP_VALUE='auto_prepend_file=/tmp/crux-root.php' \
cgi-fcgi -bind -connect 127.0.0.1:9001
```

A resposta trouxe `uid=0(root) gid=0(root)`. Depois, alterei apenas o comando PHP temporário para ler `/root/root.txt`.

A flag de root foi lida em `/root/root.txt` para confirmar a escalação; seu valor fica **[REDACTED]**.

## Cadeia resumida

```text
HTTP :80 (crux.hc)
  → BOLA em POST /api/v1/pass/renew
  → sessão da conta owner (member_id 593)
  → importação de AST Handlebars
  → execução de comandos como node
  → acesso ao php-fpm local :9001, pool wp como root
  → PHP_VALUE auto_prepend_file
  → execução como root
```

## Lições aprendidas

- Na renovação, o servidor deve vincular o passe ao usuário da sessão e conferir autorização para qualquer ID recebido.
- O endpoint de restauração deve aceitar apenas o formato esperado e validar os tipos da AST; idealmente, não deve compilar AST recebida do cliente.
- O pool php-fpm do WordPress deve rodar com usuário sem privilégios e controlar quem pode conectar ao socket. Configurações FastCGI que alteram a execução, como `PHP_VALUE`, devem ser tratadas como superfície privilegiada.

## Limpeza

O template original foi restaurado e a resposta voltou a exibir `gate-01`. O PHP temporário e os scripts temporários usados na Kali foram removidos. A conta de teste criada no portal permanece, pois não foi encontrado fluxo de exclusão de conta.
