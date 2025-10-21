# Laboratório de Pentest em VMs (Kali + Metasploitable/DVWA/SMB)

[![status](https://img.shields.io/badge/status-educacional-blue)](#aviso-ético-e-legal)
[![kali](https://img.shields.io/badge/Kali-rolling-557CFF?logo=kalilinux\&logoColor=white)](https://www.kali.org/)
[![virtualbox](https://img.shields.io/badge/VirtualBox-7.x-1f87ff?logo=virtualbox\&logoColor=white)](https://www.virtualbox.org/)
[![nmap](https://img.shields.io/badge/nmap-7.x-2ea44f)](https://nmap.org/)
[![medusa](https://img.shields.io/badge/medusa-bruteforce-orange)](#3-ftp-criação-de-wordlists--força-bruta)
[![enum4linux](https://img.shields.io/badge/enum4linux-SMB%20enum.-yellow)](#5-smb-enumeração-e-acesso)

> ## Aviso ético e legal
>
> Este projeto é **exclusivamente educacional**, feito em **ambiente controlado** (VMs locais) e com **autorização explícita**. **Nunca** use técnicas de varredura, força bruta ou exploração em sistemas de terceiros sem permissão. Você é o único responsável pelo uso seguro.

---

## Sumário

* [Objetivo](#objetivo)
* [Topologia do laboratório](#topologia-do-laboratório)
* [Requisitos](#requisitos)
* [Fluxo resumido](#fluxo-resumido)
* [Passo a passo (comandos)](#passo-a-passo-comandos)

  * [1) Descobrir IP e conectividade](#1-descobrir-ip-e-conectividade)
  * [2) Varredura de portas e versões](#2-varredura-de-portas-e-versões)
  * [3) FTP: criação de wordlists + força bruta](#3-ftp-criação-de-wordlists--força-bruta)
  * [4) Web (DVWA): parâmetros + força bruta HTTP](#4-web-dvwa-parâmetros--força-bruta-http)
  * [5) SMB: enumeração e acesso](#5-smb-enumeração-e-acesso)
* [Estrutura sugerida do repositório](#estrutura-sugerida-do-repositório)
* [Resultados (observados no lab)](#resultados-observados-no-lab)
* [Lições aprendidas](#lições-aprendidas)
* [Troubleshooting rápido](#troubleshooting-rápido)
* [Boas práticas](#boas-práticas)
* [Licença](#licença)

---

## Objetivo

Montar um mini-laboratório de segurança e praticar **reconhecimento** e **ataques de autenticação por força bruta** em serviços comuns (FTP, web/DVWA e SMB) usando **Kali Linux** como atacante e **Metasploitable** (com DVWA) como alvo.

## Topologia do laboratório

* **VM 1 (Attacker):** Kali Linux (ferramentas: `nmap`, `medusa`, `enum4linux`, etc.)
* **VM 2 (Target):** Metasploitable (imagem vulnerável do projeto Metasploit) e **DVWA**.
* **Rede:** Host-Only/Interna (ex.: `192.168.56.0/24`)

> IP do alvo usado nos exemplos: `192.168.56.102`.

## Requisitos

* Oracle **VirtualBox** 7.x
* **Kali Linux** (ISO/OVA)
* **Metasploitable** (ou equivalente vulnerável) + **DVWA** habilitado
* Rede **Host-Only**/interna entre as VMs

---

## Fluxo resumido

1. **Descobrir IP** e testar conectividade com o alvo.
2. **Varredura** de portas/serviços (`nmap -sV`).
3. **FTP:** criar **wordlists** e usar **Medusa**; validar no `ftp`.
4. **Web (DVWA):** inspecionar parâmetros do formulário e atacar via **Medusa** (HTTP).
5. **SMB:** **enumeração** com `enum4linux` → wordlists → **Medusa** (`-M smbnt`) → validar com `smbclient`.

---

## Passo a passo (comandos)

### 1) Descobrir IP e conectividade

```bash
ip a
ping -c 3 192.168.56.102
```

### 2) Varredura de portas e versões

```bash
nmap -sV -p 21,22,80,445,139 192.168.56.102
# -sV: detecta versão dos serviços
# -p : portas de interesse (FTP=21, SSH=22, HTTP=80, SMB=139/445)
```

### 3) FTP: criação de wordlists + força bruta

Crie listas simples de **usuários** e **senhas**:

```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

Ataque com **Medusa** (FTP):

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp -t 6
# -h: host alvo | -U/-P: wordlists | -M: módulo | -t: threads por host
```

Valide credenciais que retornarem `[SUCCESS]`:

```bash
ftp 192.168.56.102
# informe o user/senha encontrados; o login deve ter sucesso
```

### 4) Web (DVWA): parâmetros + força bruta HTTP

Acesse no navegador:

```
http://192.168.56.102/dvwa/login.php
```

Inspecione o formulário (DevTools) para mapear **parâmetros** e **mensagens de erro**.

Ataque com **Medusa** (HTTP) — **atenção às aspas**:

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http \
  -m "PAGE:/dvwa/login.php" \
  -m "FORM:username=^USER^&password=^PASS^&login=login" \
  -m "FAIL:Login failed" \
  -t 6
```

> Dica DVWA: se houver **token CSRF** atrapalhando, entre em *DVWA Security* e ajuste para **Low** (ambiente didático).

### 5) SMB: enumeração e acesso

Enumere amplamente e salve a saída:

```bash
enum4linux -a 192.168.56.102 | tee reports/enum4_output.txt
```

Monte **wordlists** a partir dos usuários encontrados:

```bash
echo -e "user\nmsfadmin\nservice" > wordlists/smb_users.txt
echo -e "password\n123456\nWelcome123\nmsfadmin" > wordlists/senhas_spray.txt
```

Ataque SMB com **Medusa**:

```bash
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -P wordlists/senhas_spray.txt \
  -M smbnt -t 2 -T 50
# -t: threads por host | -T: hosts em paralelo
```

Valide privilégios e compartihamentos:

```bash
smbclient -L //192.168.56.102 -U msfadmin
```

---

## Estrutura sugerida do repositório

```
.
├── README.md
├── wordlists/
│   ├── users.txt
│   ├── pass.txt
│   ├── smb_users.txt
│   └── senhas_spray.txt
└── reports/
    └── enum4_output.txt
```

---

## Resultados (observados no lab)

* **FTP:** credenciais descobertas via `medusa` → login ok no `ftp`.
* **DVWA:** sucesso na força bruta HTTP após mapear parâmetros e string de falha.
* **SMB:** `enum4linux` → wordlists → `medusa` → **ACCOUNT FOUND** (acesso com privilégio); listagem de shares com `smbclient`.

> Observação: a eficácia depende de configurações **deliberadamente vulneráveis** do alvo (Metasploitable/DVWA).

---

## Lições aprendidas

* **Reconhecimento** (serviços/versões) direciona o ataque e economiza tempo.
* Em **aplicações web**, entender **parâmetros** e **respostas** é chave para automatizar tentativas.
* **Enumeração SMB** (usuários/shares/policies) antes de tentar senhas aumenta a taxa de sucesso.
* Ajustar **paralelismo** (`-t`, `-T`) no `medusa` impacta desempenho e estabilidade.
* **Wordlists contextuais** (derivadas da enumeração) superam listas genéricas.

---

## Troubleshooting rápido

* **Sem resposta ao ping:** verifique rede Host-Only/Firewall do host.
* **`nmap` sem serviços:** confirme IP do alvo e se a VM target está ligada.
* **DVWA falhando com `medusa`:** verifique **endpoint** e **parâmetros**; ajuste *Security Level* para **Low** se houver token CSRF; confira a string de **FAIL**.
* **`medusa -M smbnt` muito lento:** reduza `-T`, mantenha `-t` baixo; valide conectividade SMB (139/445).
* **`smbclient` não lista shares:** credenciais insuficientes ou SMB bloqueado; tente outro usuário encontrado na enumeração.

---

## Boas práticas

* Mantenha o lab **isolado** (sem Internet para a VM vulnerável).
* **Documente** cada etapa (logs, saída de comandos).
* **Limite** as threads para não travar o alvo.
* **Atualize** o Kali, mas preserve a imagem vulnerável para fins didáticos.

---

## Licença

Este material é disponibilizado para fins **educacionais**. 
