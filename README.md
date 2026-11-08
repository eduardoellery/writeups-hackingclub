# Writeups — Hacking Club

Registros de máquinas de aprendizado da Hacking Club. Cada arquivo descreve o caminho que foi **validado**, as decisões tomadas durante o teste e as evidências necessárias para reproduzi-lo. Os textos contêm spoilers da exploração.

## Índice

| Máquina | Tipo | Acesso obtido | Writeup |
| --- | --- | --- | --- |
| Stick | CTF / pentest de aprendizado | Usuário e root | [stick.md](stick.md) |

## Ambiente usado

Os testes são conduzidos com **Codex no Windows**, conectado por **SSH** ao Kali para usar o **`kali-server-mcp`**. O Kali estabelece a VPN com a plataforma. Os comandos mostrados nos writeups são executados no Kali ou, após acesso inicial, na própria máquina do desafio. Essa infraestrutura é o ambiente do testador; não faz parte da exploração do alvo.

## Convenção dos arquivos

- Um repositório por plataforma: `writeups-<plataforma>`.
- Um arquivo Markdown por máquina: `<maquina>.md`, em minúsculas e com hífens quando necessário.
- O `README.md` funciona como índice. Se um writeup precisar de imagens no futuro, elas podem ficar em `assets/<maquina>/`.
- Flags, senhas, chaves, cookies e tokens são sempre apresentados como `[REDACTED]`. IPs temporários podem ser substituídos por `<IP_DA_MAQUINA>` para manter os comandos reutilizáveis.

Os testes devem respeitar o escopo e as regras da plataforma correspondente.
