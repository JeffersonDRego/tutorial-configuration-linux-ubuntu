# 🖥️ Guia: VPS com Ubuntu 26.04 LTS

> Guia completo de instalação, configuração de segurança e manutenção de uma VPS usando Ubuntu LTS.

## 📌 Índice

1. [O que é Cloud-Init?](#1-o-que-é-cloud-init)
2. [Instalação do Sistema Operacional](#2-instalação-do-sistema-operacional)
3. [Acessando o servidor pela primeira vez](#3-acessando-o-servidor-pela-primeira-vez)
4. [Pós-instalação: segurança em ordem](#4-pós-instalação-segurança-em-ordem)
   - [Atualizar o sistema](#1-atualizar-o-sistema-faça-isso-primeiro)
   - [Criar usuário não-root](#2-criar-um-usuário-não-root)
   - [Configurar o firewall (UFW)](#3-configurar-o-firewall-ufw--siga-a-ordem-exata)
   - [Atualizações automáticas de segurança](#4-ativar-atualizações-automáticas-de-segurança)
   - [Trocar a porta do SSH](#5-trocar-a-porta-do-ssh-opcional-mas-recomendado)
   - [Configurar Chave SSH e desativar senhas](#6-configurar-autenticação-por-chave-ssh-e-desativar-senhas)
   - [Instalar Fail2Ban](#7-configurar-fail2ban-proteção-ativa)
5. [Manutenção: como manter o servidor atualizado](#5-manutenção-como-manter-o-servidor-atualizado)
6. [Auditoria e Testes de Segurança (Nmap e Lynis)](#6-auditoria-e-testes-de-segurança)
7. [Monitorando Vulnerabilidades no Ubuntu 26.04 LTS](#7-monitorando-vulnerabilidades-no-ubuntu-2604-lts)

---

## 1. O que é Cloud-Init?

**Cloud-Init** é o padrão da indústria usado por praticamente todos os provedores de cloud (AWS, DigitalOcean, Hetzner, Hostinger, etc.) para inicializar e configurar uma máquina virtual no seu primeiro boot de forma automatizada. Pense nele como um "setup automático" que a máquina executa sozinha logo que liga.

Quando você seleciona opções de instalação com "1-clique" no painel do seu provedor (como "Ubuntu + Docker" ou "Ubuntu + Painel Web"), o que acontece nos bastidores é o Cloud-Init em ação. O provedor sobe uma imagem limpa do Ubuntu e injeta um script invisível que cria usuários, cadastra chaves SSH, atualiza o sistema e instala as ferramentas escolhidas.

**Pode ser o seu caso:** se você usou uma dessas imagens pré-configuradas oferecidas pelo seu provedor, grande parte do setup inicial já foi feito automaticamente nos primeiros minutos em que a máquina ligou.

No entanto, o foco deste guia é ensinar a configuração de uma **VPS Ubuntu limpa**. O objetivo é tirar a "caixa preta" da jogada e te dar controle total. Fazendo isso manualmente, você entende exatamente quais portas estão abertas, quais usuários existem e como o firewall está operando desde o segundo zero.

---

## 2. Instalação do Sistema Operacional

Ao criar sua VPS em provedores de cloud, escolha a imagem limpa do **Ubuntu 26.04 LTS**.

> ✅ Essa é a base ideal para começar a configurar seu ambiente de produção de forma segura, com suporte de longo prazo.

---

## 3. Acessando o servidor pela primeira vez

Você vai acessar a VPS via **SSH** — é como o terminal da sua máquina, mas conectado remotamente ao servidor.

O seu provedor de cloud vai te fornecer o **IP da VPS** e a **credencial de acesso** (senha ou chave SSH) gerada durante a criação da máquina.

```bash
ssh root@IP_DA_SUA_VPS
```

Na primeira conexão, o terminal vai perguntar se você confia no servidor. Responda `yes`.

> 💡 No Windows, use o **PowerShell** ou instale o **Windows Terminal**. No Mac/Linux, use o terminal nativo.

---

## 4. Pós-instalação: segurança em ordem

> 💡 **Sobre o `sudo`:** os comandos abaixo assumem que você está logado como **root** (logo após a primeira conexão). Depois de criar seu usuário não-root e trocar para ele, todos os comandos que modificam o sistema precisam do prefixo `sudo`. Regra simples:
> - Comando que **lê** algo → sem sudo (ex: `ufw status`)
> - Comando que **modifica** algo → com sudo (ex: `sudo ufw enable`)

---

### 1. Atualizar o sistema (faça isso primeiro)

Esse é o **primeiro comando que você deve rodar em qualquer servidor novo**, sem exceção. A imagem usada na instalação pode estar desatualizada — este comando aplica todos os patches de segurança disponíveis.

> ⚠️ Durante o `apt upgrade`, pode aparecer uma tela perguntando o que fazer com configurações modificadas. Na dúvida, selecione **"manter a versão local atualmente instalada"**.

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

> ⚠️ **ATENÇÃO: ordem importa aqui.** Se você ativar o firewall ANTES de liberar a porta do SSH, vai perder o acesso ao servidor e terá que usar o console de emergência do seu provedor para recuperar.

**Siga exatamente esta sequência:**

**Passo 1 — Libere o SSH ANTES de qualquer coisa:**
```bash
sudo ufw allow ssh
```
> Isso libera a porta 22 (SSH padrão).

**Passo 2 — Libere as portas necessárias para a sua aplicação:**
```bash
sudo ufw allow 80     # HTTP
sudo ufw allow 443    # HTTPS
# Adicione outras portas específicas que sua aplicação precisar
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
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

> ✅ Se o SSH (porta 22) aparece na lista como ALLOW, você está seguro. O firewall está ativo e você não perdeu o acesso.

**Passo 5 — Se aparecerem regras duplicadas, limpe:**
Dependendo da instalação, podem haver regras duplicadas (ex: porta 80 aparecendo duas vezes — uma com `/tcp` e uma sem). Isso não causa problema de segurança, mas é bom manter organizado:

```bash
# Remove as entradas duplicadas (sem /tcp)
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

**Por que Yes é a escolha certa aqui:** o `unattended-upgrades` não instala tudo — ele aplica apenas patches do repositório `security`, que são exclusivamente correções de vulnerabilidades já validadas. Não são versões novas de software, não quebram funcionalidades. Uma vulnerabilidade crítica descoberta hoje pode estar sendo explorada ativamente — cada dia sem o patch é um dia de risco desnecessário.

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

> ⚠️ **Atenção — Socket Activation.** Nas versões mais recentes do Ubuntu, o SSH é gerenciado pelo systemd via socket, então apenas `systemctl restart ssh` não é suficiente. Você precisa rodar os dois comandos abaixo:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

**Passo 5 — Abra uma NOVA aba/janela do terminal e teste a conexão APÓS o restart:**
```bash
ssh -p 2222 seuusuario@IP_DA_SUA_VPS
```

> ⚠️ Só feche a sessão antiga depois de confirmar que a nova funciona. Se algo der errado, você ainda tem a sessão aberta para corrigir.

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

## 5. Manutenção: como manter o servidor atualizado

### Verificar o que está disponível para atualizar
```bash
sudo apt update
sudo apt list --upgradable
```

### Aplicar todas as atualizações
```bash
sudo apt upgrade -y
```

### Limpar pacotes antigos e cache desnecessário
```bash
sudo apt autoremove -y && sudo apt autoclean
```

### Verificar se o servidor precisa ser reiniciado após updates
```bash
cat /var/run/reboot-required
```
> Se esse arquivo existir, é porque uma atualização crítica (como de kernel) foi aplicada e requer reboot. Agende para um momento de baixo tráfego.

**Frequência recomendada:** semanalmente, ou sempre que sair uma notícia grave sobre segurança em servidores Linux.

---

## 6. Auditoria e Testes de Segurança

Para garantir que o servidor está protegido e que nenhuma porta indesejada foi exposta externamente (especialmente por regras automáticas de ferramentas como Docker), execute os testes abaixo.

### 1. Varredura de Portas Externa (Executar na máquina local)
Utilize o **Nmap** em seu ambiente local para escanear a VPS.

```bash
# Instale nmap no seu S.O. local
sudo apt install nmap -y

# Varredura furtiva completa em todas as 65.535 portas
sudo nmap -sS -Pn -p- IP_DA_SUA_VPS
```
**Resultado esperado:** Apenas as portas de produção (ex: 80, 443) e a nova porta SSH (2222) devem constar como `open`. Todas as demais devem constar como `filtered` (no-response).

### 2. Auditoria Interna do Sistema (Executar dentro da VPS)
O Lynis é uma excelente ferramenta para analisar configurações incorretas e permissões de arquivos.

```bash
sudo apt install lynis -y
sudo lynis audit system
```
Foque em corrigir os avisos em vermelho (WARNING) do relatório gerado.

> ⚠️ **Aviso de Atenção sobre Docker & Firewall:** O Docker manipula o `iptables` diretamente e ignora as regras padrão do UFW. Ao subir serviços em contêineres Docker, certifique-se de não expor portas de bancos de dados (como 5432, 3306) diretamente para o Host (`0.0.0.0`). Deixe que proxies reversos realizem o roteamento pelas portas públicas padrão (80/443).

---

## 7. Monitorando Vulnerabilidades no Ubuntu 26.04 LTS

O Ubuntu 26.04 LTS é uma versão de Long Term Support, o que significa que receberá atualizações de manutenção e patches de segurança contínuos até **abril de 2031**.

Por ser um sistema altamente moderno, ele já compila seus binários principais com camadas extras de proteção (como `FORTIFY_SOURCE=3`), mitigando proativamente classes inteiras de ataques, como buffer overflows.

Ainda assim, vulnerabilidades (CVEs) são descobertas regularmente na comunidade de software livre. A maioria dessas falhas em ambientes de servidor costuma ser de **escalada de privilégio local** — ou seja, o atacante já precisaria ter algum acesso inicial restrito à máquina para conseguir se tornar `root`.

### Como manter a segurança em dia:

1. **Confie no `unattended-upgrades`**: Como configuramos no passo 4, ele garantirá que correções críticas validadas pela Canonical cheguem ao seu servidor quase imediatamente.
2. **Monitore fontes oficiais**:
   - Notícias de Segurança: https://ubuntu.com/security/notices
   - Base de Dados de CVEs: https://ubuntu.com/security/cves

---

## ✅ Checklist do Dia 1

```
[ ] Acessar via SSH com a credencial do provedor
      ssh root@IP_DA_VPS

[ ] Atualizar o sistema
      apt update && apt upgrade -y

[ ] Reiniciar e reconectar
      reboot

[ ] Criar usuário não-root com sudo
      adduser seuusuario
      usermod -aG sudo seuusuario
      su - seuusuario

[ ] Configurar firewall — NESSA ORDEM:
      sudo ufw allow ssh       ← SSH PRIMEIRO, sempre
      sudo ufw allow 80        ← HTTP
      sudo ufw allow 443       ← HTTPS
      sudo ufw enable          ← SÓ ENTÃO ativar
      sudo ufw status          

[ ] Configurar unattended-upgrades
      sudo apt install unattended-upgrades -y
      sudo dpkg-reconfigure --priority=low unattended-upgrades
      (Selecionar "Yes")

[ ] Trocar porta do SSH para 2222
      sudo nano /etc/ssh/sshd_config  → Port 2222
      sudo ufw allow 2222
      sudo systemctl daemon-reload
      sudo systemctl restart ssh.socket
      (Testar em nova janela: ssh -p 2222 seuusuario@IP)
      sudo ufw delete allow ssh

[ ] Configurar autenticação por Chave SSH e desativar senhas
      # Na sua máquina local:
      ssh-keygen -t ed25519
      ssh-copy-id -p 2222 seuusuario@IP_DA_VPS
      
      # Na VPS:
      sudo nano /etc/ssh/sshd_config
      → Mudar PasswordAuthentication para no
      → Mudar PermitRootLogin para no
      
      sudo systemctl restart ssh.socket
      (ATENÇÃO: Teste em nova janela antes de fechar a atual)

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

- [Ubuntu Security Notices](https://ubuntu.com/security/notices)
- [Ubuntu 26.04 LTS Release Notes](https://documentation.ubuntu.com/release-notes/26.04/)
- [Guia UFW no Ubuntu](https://help.ubuntu.com/community/UFW)