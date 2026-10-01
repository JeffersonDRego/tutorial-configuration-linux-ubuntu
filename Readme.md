# 🖥️ Guia: VPS com Ubuntu 24.04 LTS + EasyPanel

> Guia completo de instalação, configuração de segurança e manutenção de uma VPS na Locaweb usando Ubuntu 24.04 LTS com EasyPanel.

---

## 📌 Índice

1. [O que é Cloud-Init?](#o-que-é-cloud-init)
2. [A instalação automática (Locaweb + EasyPanel)](#a-instalação-automática)
3. [Acessando o servidor pela primeira vez](#acessando-o-servidor-pela-primeira-vez)
4. [Pós-instalação: segurança em ordem](#pós-instalação-segurança-em-ordem)
   - [Atualizar o sistema](#1-atualizar-o-sistema-faça-isso-primeiro)
   - [Criar usuário não-root](#2-criar-um-usuário-não-root)
   - [Configurar o firewall (UFW)](#3-configurar-o-firewall-ufw--siga-a-ordem-exata)
   - [Atualizações automáticas de segurança](#4-ativar-atualizações-automáticas-de-segurança)
   - [Trocar a porta do SSH](#5-trocar-a-porta-do-ssh-opcional-mas-recomendado)
   - [Configurar Chave SSH e desativar senhas](#6-configurar-autenticação-por-chave-ssh-e-desativar-senhas)
   - [Instalar Fail2Ban](#7-configurar-fail2ban-proteção-ativa)
5. [Manutenção: como manter o servidor atualizado](#manutenção-como-manter-o-servidor-atualizado)
6. [Auditoria e Testes de Segurança (Nmap e Lynis)](#-auditoria-e-testes-de-segurança)
7. [Vulnerabilidades conhecidas no Ubuntu 24.04...](#vulnerabilidades-conhecidas...)

---

## O que é Cloud-Init?

**Cloud-Init** é um script que roda automaticamente na **primeira inicialização** da VPS, antes de você sequer fazer login. Pense nele como um "setup automático" que a máquina executa sozinha logo que liga.

Com ele você pode criar usuários, instalar pacotes, configurar o firewall — tudo de forma automatizada e sem intervenção manual.

**No seu caso: você não precisa usar isso.** A instalação da Locaweb com EasyPanel já vai cuidar do setup inicial automaticamente. A opção existe para casos avançados onde você quer personalizar a máquina desde o primeiro boot.

---

## A instalação automática

Ao escolher **EasyPanel com Ubuntu 24.04 LTS** no painel da Locaweb, o provedor vai:

1. Instalar o Ubuntu 24.04 LTS limpo
2. Rodar o script oficial do EasyPanel automaticamente
3. Deixar o painel acessível via `http://IP_DA_SUA_VPS:3000`

> ✅ Essa é a melhor opção para você agora. Avance com ela.

---

## Acessando o servidor pela primeira vez

Você vai acessar a VPS via **SSH** — é como o terminal da sua máquina, mas conectado remotamente ao servidor.

A Locaweb vai te fornecer o **IP da VPS** e você usará a **senha que cadastrou** durante a instalação.

```bash
ssh root@IP_DA_SUA_VPS
```

Na primeira conexão, o terminal vai perguntar se você confia no servidor. Responda `yes`.

> 💡 No Windows, use o **PowerShell** ou instale o **Windows Terminal**. No Mac/Linux, use o terminal nativo.

---

## Pós-instalação: segurança em ordem

> 💡 **Sobre o `sudo`:** os comandos abaixo assumem que você está logado como **root** (logo após a primeira conexão). Depois de criar seu usuário não-root e trocar para ele, todos os comandos que modificam o sistema precisam do prefixo `sudo`. Regra simples:
> - Comando que **lê** algo → sem sudo (ex: `ufw status`)
> - Comando que **modifica** algo → com sudo (ex: `sudo ufw enable`)

---

### 1. Atualizar o sistema (faça isso primeiro)

Esse é o **primeiro comando que você deve rodar em qualquer servidor novo**, sem exceção. A imagem usada na instalação pode estar desatualizada — este comando aplica todos os patches de segurança disponíveis.

> ⚠️ Durante o `apt upgrade`, pode aparecer uma tela roxa perguntando o que fazer com o arquivo `sshd_config` modificado. Selecione **"manter a versão local atualmente instalada"** — a instalação automática já configurou o SSH corretamente e você não quer sobrescrever isso.

```bash
apt update && apt upgrade -y
```

- `apt update` → atualiza a lista de pacotes disponíveis (como um `npm install` sem instalar nada ainda)
- `apt upgrade -y` → instala as atualizações disponíveis (o `-y` confirma tudo automaticamente)

Após a atualização, **reinicie o servidor** para aplicar as atualizações do kernel:

```bash
reboot
```

Aguarde ~30 segundos e reconecte via SSH.

---

### 2. Criar um usuário não-root

Trabalhar direto como `root` é arriscado — qualquer comando errado tem poder total para destruir o sistema. Crie um usuário normal com permissões de `sudo`:

```bash
# Cria o usuário (substitua "seuusuario" pelo nome que quiser)
adduser seuusuario

# Adiciona o usuário ao grupo sudo
usermod -aG sudo seuusuario
```

> 💡 `sudo` funciona como o "Executar como Administrador" do Windows. Com ele, você executa comandos poderosos apenas quando necessário, e o sistema registra o que foi feito.

Para testar, troque para o novo usuário:

```bash
su - seuusuario
```

---

### 3. Configurar o firewall (UFW) — siga a ordem exata

> ⚠️ **ATENÇÃO: ordem importa aqui.** Se você ativar o firewall ANTES de liberar a porta do SSH, vai perder o acesso ao servidor e terá que usar o console de emergência da Locaweb para recuperar.

**Siga exatamente esta sequência:**

**Passo 1 — Libere o SSH ANTES de qualquer coisa:**
```bash
sudo ufw allow ssh
```
> Isso libera a porta 22 (SSH padrão). Se aparecer "Skipping adding existing rule", é porque a instalação automática já criou essa regra — tudo certo, continue.

**Passo 2 — Libere as portas necessárias para o EasyPanel:**
```bash
sudo ufw allow 3000   # Painel do EasyPanel
sudo ufw allow 80     # HTTP
sudo ufw allow 443    # HTTPS
```

**Passo 3 — SÓ AGORA ative o firewall:**
```bash
sudo ufw enable
```

**Passo 4 — Verifique se está tudo certo:**
```bash
sudo ufw status
```

A saída deve ser parecida com esta:

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
3000/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

> ✅ Se o SSH (porta 22) aparece na lista como ALLOW, você está seguro. O firewall está ativo e você não perdeu o acesso.

**Passo 5 — Se aparecerem regras duplicadas, limpe:**

A instalação automática pode ter criado regras que se somam às que você adicionou, deixando duplicatas (ex: porta 80 aparecendo duas vezes — uma com `/tcp` e uma sem). Isso não causa problema de segurança, mas é bom manter organizado:

```bash
# Remove as entradas duplicadas (sem /tcp)
sudo ufw delete allow 3000
sudo ufw delete allow 80
sudo ufw delete allow 443

# Confirme o resultado limpo
sudo ufw status
```

---

### 4. Ativar atualizações automáticas de segurança

Para não depender de lembrar de rodar `apt upgrade`, instale e configure o pacote de atualizações automáticas:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Na tela interativa que aparecer, selecione **"Yes"**.

**Por que Yes é a escolha certa aqui:** o `unattended-upgrades` não instala tudo — ele aplica apenas patches do repositório `security`, que são exclusivamente correções de vulnerabilidades já validadas pela Canonical. Não são versões novas de software, não quebram funcionalidades. Uma vulnerabilidade crítica descoberta hoje pode estar sendo explorada ativamente — cada dia sem o patch é um dia de risco desnecessário.

Isso vai aplicar **patches de segurança automaticamente** em segundo plano, sem te incomodar e sem derrubar serviços.

---

### 5. Trocar a porta do SSH (opcional, mas recomendado)

A porta padrão 22 é constantemente varrida por bots ao redor do mundo tentando invadir servidores. Trocar para outra porta reduz esse ruído drasticamente.

**Passo 1 — Edite o arquivo de configuração do SSH:**
```bash
sudo nano /etc/ssh/sshd_config
```

**Passo 2 — Encontre a linha `Port 22` e troque o número:**
```
Port 2222
```
> Salve com `Ctrl+O` → `Enter` → `Ctrl+X`

**Passo 3 — ANTES de reiniciar o SSH, libere a nova porta no firewall:**
```bash
sudo ufw allow 2222
```

**Passo 4 — Reinicie o SSH:**

> ⚠️ **Atenção — Ubuntu 24.04 usa socket activation.** Nesta versão, o SSH é gerenciado pelo systemd via socket, então apenas `systemctl restart ssh` não é suficiente. Você precisa rodar os dois comandos abaixo:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

**Passo 5 — Abra uma NOVA aba/janela do terminal e teste a conexão APÓS o restart:**
```bash
ssh -p 2222 seuusuario@IP_DA_SUA_VPS
```

> ⚠️ Só feche a sessão antiga depois de confirmar que a nova funciona. Se algo der errado, você ainda tem a sessão aberta para corrigir.
>
> 💡 Testar ANTES do restart vai dar timeout — isso é normal. A porta 2222 só responde depois que o serviço é reiniciado.

**Passo 6 — Após confirmar que funciona, remova a porta 22 do firewall:**
```bash
sudo ufw delete allow ssh
sudo ufw status
```

A partir de agora, sempre conecte assim:
```bash
ssh -p 2222 seuusuario@IP_DA_SUA_VPS
```

---

## Manutenção: como manter o servidor atualizado

### Verificar o que está disponível para atualizar
```bash
apt update
apt list --upgradable
```

### Aplicar todas as atualizações
```bash
apt upgrade -y
```

### Atualizar pacotes de sistema mais profundos (ocasionalmente)
```bash
apt full-upgrade -y
```

### Limpar pacotes antigos e cache desnecessário
```bash
apt autoremove -y && apt autoclean
```

### Verificar se o servidor precisa ser reiniciado após updates
```bash
cat /var/run/reboot-required
```
> Se esse arquivo existir, é porque uma atualização de kernel foi aplicada e requer reboot. Agende para um momento de baixo tráfego.

**Frequência recomendada:** semanalmente. Ou imediatamente sempre que você ouvir sobre uma vulnerabilidade crítica no Linux.

---

## 🛡️ Auditoria e Testes de Segurança

Para garantir que o servidor está realmente protegido e que nenhuma porta indesejada foi exposta externamente (especialmente por regras automáticas do Docker/EasyPanel), execute os testes abaixo.

### 1. Varredura de Portas Externa (Executar na máquina local)
Utilize o **Nmap** em seu ambiente local para escanear a VPS. O objetivo é validar que o firewall está descartando requisições nas portas invisíveis.

```bash
# Instale nmap no seu S.O.
sudo apt install nmap -y
```

```bash
# Varredura furtiva completa em todas as 65.535 portas
sudo nmap -sS -Pn -p- IP_DA_SUA_VPS
```
Resultado esperado: Apenas as portas de produção (80, 443), a porta administrativa do EasyPanel (3000) e a nova porta SSH (2222) devem constar como open. Todas as demais devem constar como filtered (no-response).

### 2. Auditoria Interna do Sistema (Executar dentro da VPS)

O Lynis é uma ferramenta de auditoria de segurança para sistemas Linux que analisa configurações incorretas, permissões de arquivos e atualizações pendentes.

```bash
sudo apt install lynis -y
sudo lynis audit system
```

Uso: Analise o relatório gerado no terminal. Foque em corrigir os avisos em vermelho (WARNING) e analise as sugestões (SUGGESTIONS) para blindar ainda mais o kernel e os binários do sistema.


---

### ⚠️ Nota importante sobre o Docker vs UFW (Dica de Atenção)
Caso queira deixar um aviso explícito sobre o comportamento do Docker no guia para não esquecer no futuro, você pode colocar este bloco de texto logo abaixo da seção do EasyPanel:

```markdown
> ⚠️ **Aviso de Atenção sobre Docker & Firewall:** O Docker manipula o `iptables` diretamente e ignora as regras padrão do UFW. Ao subir serviços no EasyPanel, certifique-se de não expor portas de bancos de dados (como 5432, 3306) diretamente para o Host (0.0.0.0). Deixe que o Proxy interno do painel realize o roteamento pelas portas públicas padrão (80/443).
```

---

## Vulnerabilidades conhecidas no Ubuntu 24.04 LTS

O Ubuntu 24.04 LTS é uma versão LTS (Long Term Support) — ou seja, receberá suporte e patches de segurança até **abril de 2029**. Isso é ótimo para um servidor de produção.

Porém, como todo sistema ativo, ele possui CVEs (vulnerabilidades catalogadas). As mais relevantes recentemente:

| CVE | Severidade | Descrição | Status |
|-----|-----------|-----------|--------|
| CVE-2026-3888 | Alta (7.8) | Escalada de privilégio local via `snap-confine` + `systemd-tmpfiles`. Um usuário local comum pode virar root. | ✅ Corrigido via `apt upgrade` |
| CVE-2025-32462/32463 | Alta | Falhas críticas no `sudo` permitindo escalada de privilégio local. | ✅ Corrigido via `apt upgrade` |
| CVE-2025-38352 | Média | Falha no kernel Linux que pode causar crash ou escalada de privilégio. | ✅ Corrigido via `apt upgrade` |

> 💡 **Traduzindo:** a maioria dessas falhas são de "escalada de privilégio local" — ou seja, um atacante precisaria **já estar dentro da sua máquina** com um usuário comum para explorá-las. Em uma VPS bem configurada com acesso restrito, o risco real é baixo. Mesmo assim, mantenha o sistema atualizado.

### Ponto positivo do 24.04 LTS

Esta versão foi compilada com proteções mais rígidas que as anteriores. Usa `FORTIFY_SOURCE=3` (vs. `=2` do 22.04), o que aumenta significativamente a detecção e mitigação de buffer overflow — uma das formas mais comuns de exploração.

### Como monitorar novas vulnerabilidades

- Site oficial: https://ubuntu.com/security/notices
- CVEs ativos: https://ubuntu.com/security/cves

---

## ✅ Checklist do Dia 1

```
[ ] Acessar via SSH com a senha cadastrada na Locaweb
      ssh root@IP_DA_VPS

[ ] Na primeira conexão, responder "yes" para confiar no servidor

[ ] Atualizar o sistema
      apt update && apt upgrade -y
      (Se aparecer tela do sshd_config → "manter a versão local")

[ ] Reiniciar e reconectar
      reboot

[ ] Criar usuário não-root com sudo
      adduser seuusuario
      usermod -aG sudo seuusuario
      su - seuusuario
      (A partir daqui, use sudo em todos os comandos que modificam o sistema)

[ ] Configurar firewall — NESSA ORDEM:
      sudo ufw allow ssh       ← SSH PRIMEIRO, sempre
      sudo ufw allow 3000      ← EasyPanel
      sudo ufw allow 80        ← HTTP
      sudo ufw allow 443       ← HTTPS
      sudo ufw enable          ← SÓ ENTÃO ativar
      sudo ufw status          ← confirmar que 22/tcp aparece como ALLOW
      (Se houver duplicatas, limpar com sudo ufw delete allow 80/443/3000)

[ ] Configurar unattended-upgrades
      sudo apt install unattended-upgrades -y
      sudo dpkg-reconfigure --priority=low unattended-upgrades
      (Selecionar "Yes" na tela interativa)

[ ] Trocar porta do SSH para 2222
      sudo nano /etc/ssh/sshd_config  → Port 2222
      sudo ufw allow 2222
      sudo systemctl daemon-reload
      sudo systemctl restart ssh.socket
      (Testar em nova janela: ssh -p 2222 seuusuario@IP)
      sudo ufw delete allow ssh

[ ] Acessar EasyPanel em http://IP_DA_VPS:3000 e finalizar configuração

[ ] Configurar autenticação por Chave SSH e desativar senhas
      # Na sua máquina local (ex: WSL, Linux ou Terminal Mac):
      ssh-keygen -t ed25519
      ssh-copy-id -p 2222 seuusuario@IP_DA_VPS
      
      # Acesse a VPS e desative o login por senha tradicional:
      sudo nano /etc/ssh/sshd_config
      → Mudar PasswordAuthentication para no
      → Mudar PermitRootLogin para no
      
      sudo systemctl restart ssh.socket
      (ATENÇÃO: Não feche o terminal atual! Abra uma nova janela e teste o acesso `ssh -p 2222 seuusuario@IP` para garantir que a chave funciona antes de deslogar)

[ ] Configurar Fail2Ban (Proteção ativa contra força bruta)
      sudo apt install fail2ban -y
      sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
      sudo nano /etc/fail2ban/jail.local
      → Na seção [sshd], alterar a linha 'port' para 2222
      sudo systemctl enable fail2ban
      sudo systemctl start fail2ban
```

---



## 🔗 Referências

- [Documentação oficial do EasyPanel](https://easypanel.io/docs)
- [Ubuntu Security Notices](https://ubuntu.com/security/notices)
- [Ubuntu 24.04 LTS Release Notes](https://documentation.ubuntu.com/release-notes/24.04/)
- [Guia UFW no Ubuntu](https://help.ubuntu.com/community/UFW)