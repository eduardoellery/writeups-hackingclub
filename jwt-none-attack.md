# JWT None Attack — bypass de autenticação administrativa

| Campo | Valor |
| --- | --- |
| Plataforma | Hacking Club |
| Máquina | JWT None Attack |
| Tipo | Desafio |
| Data do teste | 5 de novembro de 2026 |
| Resultado | Acesso à mensagem administrativa e à flag |

> **Spoilers:** este writeup descreve a exploração completa. A flag, a senha de teste e o JWT foram ocultados.

## Ambiente de teste

Usei Codex no Windows e as ferramentas do `kali-server-mcp` para executar os testes no Kali, conectado à VPN da Hacking Club. O SSH era apenas o caminho entre Windows e Kali; não havia serviço SSH aberto na máquina do desafio. O IP temporário do alvo está representado por `<IP_DA_MAQUINA>`.

## Reconhecimento

Uma varredura TCP das portas 1 a 65535 encontrou somente `8000/tcp`, identificado como HTTP com Node.js/Express. A página inicial tinha um formulário de autenticação. Seu JavaScript enviava `username` e `password` em JSON para `POST /api/login` e, em caso de sucesso, salvava o token retornado no `localStorage`. A página `/home.html` enviava esse token no cabeçalho `Authorization` ao consultar `GET /api/home`.

```bash
nmap -sV -Pn -p 1-10000 <IP_DA_MAQUINA>
nmap -sV -Pn -p 10001-65535 --min-rate 2000 <IP_DA_MAQUINA>
```

Uma busca de caminhos encontrou `/api`, `/home.html` e `/index.html`. A rota `/api` respondia `403` sem autenticação.

## Da conta comum ao JWT

Uma combinação simples de credenciais de teste permitiu autenticar uma conta `guest`. A resposta trouxe um JWT com cabeçalho `alg: HS256` e payload que continha `name: guest` e `admin: false`. O valor da senha e o token completo ficam **[REDACTED]**.

```http
POST /api/login HTTP/1.1
Host: <IP_DA_MAQUINA>:8000
Content-Type: application/json

{"username":"guest","password":"[REDACTED]"}
```

Com esse token, `GET /api/home` respondeu que era necessário ser administrador para ver a flag. Isso confirmou que a conta funcionava e que a autorização dependia da informação de administrador associada ao token.

## Bypass com `alg: none`

Testei um JWT com cabeçalho `{"typ":"JWT","alg":"none"}`, payload `{"name":"guest","admin":true}` e assinatura vazia. Cada uma das duas partes JSON foi codificada em Base64URL e concatenada com pontos, mantendo o ponto final da assinatura vazia.

```text
base64url({"typ":"JWT","alg":"none"})
.
base64url({"name":"guest","admin":true})
.
```

Enviei o token no cabeçalho `Authorization`, no mesmo formato usado pela aplicação:

```http
GET /api/home HTTP/1.1
Host: <IP_DA_MAQUINA>:8000
Authorization: [REDACTED]
```

O endpoint retornou `200 OK`, `success: true` e uma mensagem dirigida ao administrador com a flag **[REDACTED]**. A aceitação de um token sem assinatura válida permitiu alterar `admin` de `false` para `true` e acessar conteúdo restrito.

## Cadeia resumida

```text
HTTP na porta 8000
  → login de conta guest
  → JWT com admin=false
  → JWT forjado com alg=none e admin=true
  → /api/home revela a flag administrativa [REDACTED]
```

## Lições aprendidas

- O servidor deve fixar os algoritmos de assinatura aceitos e rejeitar `alg: none` para tokens de autenticação.
- A autorização administrativa deve depender de um token cuja assinatura, validade e emissor tenham sido verificados no servidor.

Os testes foram realizados apenas na máquina do desafio.
