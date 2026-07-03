# Manual do Vídeo: Cluster Docker Swarm na Hostinger

Este documento é um registro fiel e passo a passo de todas as etapas executadas
no vídeo para subir o cluster Docker Swarm.

> **Nota:** Este guia usa o cenário do vídeo como referência: 3 nós
> (`kvm2`,`kvm4`,`kvm8`) e domínios públicos.

> **Escopo Real (sem amarras):** Você pode usar qualquer provedor (Hostinger,
> outro VPS, ou até VM local). Para seguir o fluxo completo com acesso público
> (Traefik + HTTPS + webhook), o mínimo é **1 servidor Linux com Docker + 1
> domínio apontado para ele**.
>
> **Caminho canônico do vídeo:** Para reproduzir exatamente os passos e nomes
> deste guia, use **3 nós** (`kvm2`, `kvm4`, `kvm8`).
>
> **Regra prática:** Sempre que aparecer "Execute em TODAS as VPSs", execute em
> todos os nós que você tiver. Se tiver só 1 servidor, execute uma única vez.

## Patrocínio (Hostinger)

Este projeto e o vídeo foram patrocinados pela Hostinger.

- Link: [hostinger.com/otaviomiranda](http://hostinger.com/otaviomiranda)
- Cupom: `OTAVIOMIRANDA`
- Benefício: `OTAVIOMIRANDA` dá **+10%** de desconto extra sobre o desconto do
  link.

## 0. Preparação e Desmontagem (Opcional)

Caso você já tenha um cluster rodando e queira remover um nó de forma segura
para formatá-lo (como estamos fazendo com o `kvm2`), siga estes passos. Isso
garante que os serviços sejam migrados para outros nós antes do desligamento.

**No nó gerenciador principal (ex: `kvm8`):**

1. **Drenar o nó (`Drain`):** Isso remove todos os contêineres em execução neste
   nó e os move para outros nós disponíveis. Também impede que novas tarefas
   sejam agendadas nele.

   ```bash
   docker node update kvm2 --availability drain
   ```

2. **Rebaixar para Worker (`Demote`):** Se o nó for um Manager, é uma boa
   prática rebaixá-lo para Worker antes de removê-lo. Isso ajuda a manter o
   quórum do Swarm estável.
   ```bash
   docker node demote kvm2
   ```

**No nó que será removido (ex: `kvm2`):**

3. **Sair do Cluster:** Este comando remove o nó do Swarm e limpa o estado local
   do Docker Swarm.
   ```bash
   docker swarm leave
   ```

> **Nota:** Repita este mesmo processo para outros nós secundários (como o
> `kvm4`), deixando o nó principal (`kvm8`) por último.

**De volta ao nó principal (ex: `kvm8`):**

4. **Remover metadados dos nós antigos:** Após os nós saírem, eles ficam
   listados como "Down". Precisamos removê-los da lista do gerenciador.

   ```bash
   docker node rm kvm4
   docker node rm kvm2
   ```

5. **Verificar o estado do Cluster:** Neste momento, o `kvm8` é o único nó
   restante. Ele é crítico pois segura nosso Banco de Dados, o NFS e é a porta
   de entrada (Traefik).

   Confira se os nós sumiram e o estado atual dos serviços:

   ```bash
   # Deve listar apenas o kvm8 como ativo (Ready/Active)
   docker node ls

   # Verifique suas stacks
   docker stack ls

   # Verifique onde os serviços estão rodando (agora tudo deve estar tentando ir para o kvm8 ou falhando se não houver recursos)
   docker stack services dockerswarmp1
   ```

6. **Remover a Stack:** Agora que verificamos tudo, vamos remover a stack (o
   conjunto de aplicações) para garantir que nada fique pendurado ou escrevendo
   no disco enquanto desligamos.

   ```bash
   docker stack rm dockerswarmp1
   ```

   _(Aguarde alguns instantes para que os containers sejam parados)_

7. **Destruir o Cluster (Swarm Leave Force):** Como este é o último gerenciador
   (Leader), ele não pode simplesmente "sair" (`leave`). Precisamos forçar o
   encerramento do cluster. **Atenção: Isso apaga todas as configurações do
   Swarm, segredos e serviços.**

   ```bash
   docker swarm leave --force
   ```

   **Pronto!** O cluster foi desmontado. Agora as máquinas são apenas VPSs
   comuns com Docker instalado (ou prontas para serem formatadas).

## 1. Formatação (Reset Total)

Para garantir um ambiente limpo, formatamos todas as 3 VPSs (`kvm2`, `kvm4`,
`kvm8`) usando o painel da Hostinger.

> **Se você não usa Hostinger:** Faça o procedimento equivalente no seu provedor
> (ou recrie a VM local) para começar com um Linux limpo com Docker instalado.

**No hPanel:**

1. Acesse **VPS** > **Gerenciar** (em cada nó).
2. Vá em **SO e Painel** > **Sistema Operacional** > **Mudar SO**.
3. Escolha **SO com Aplicativo** e procure por **Docker** (Ubuntu 24.04).
4. Defina uma senha forte para o `root`.

> **Nota:** Existe um vídeo anterior detalhando exaustivamente este processo de
> criação de VPS. Aqui, usamos a imagem pronta "Ubuntu 24.04 with Docker" para
> ganhar tempo e garantir que o Docker Engine já venha instalado e configurado
> corretamente.

## 2. Configuração de Hostname e DNS

Acesse cada VPS via SSH (inicialmente como `root`, usando a senha definida na
formatação) e configure a identidade da máquina.

> **Dica de Acesso Rápido:** No próprio hPanel, existe um botão **Terminal** (no
> topo direito da gestão da VPS). Ele abre um console web já logado como `root`
> (sem precisar de senha ou chave SSH configurada). É extremamente útil para
> esses ajustes iniciais antes de configurarmos o nosso acesso SSH definitivo.

**Exemplo no `kvm2`:**

1.  **Definir o Hostname:**

    ```bash
    hostnamectl set-hostname kvm2
    ```

2.  **Ajustar Hosts:** Edite o arquivo para associar o IP local ao novo nome e
    domínio (FQDN).
    ```bash
    vim /etc/hosts
    ```
    Alterar a linha `127.0.1.1` (ou similar) para ficar assim:
    ```text
    127.0.1.1       kvm2.inprod.cloud       kvm2
    ```

> **Dica Hostinger (Alternativa):** Você também pode configurar o Hostname
> diretamente pelo hPanel em **VPS > Configurações > Configurações de VPS**. O
> painel valida se o domínio realmente pertence a você (ou aponta para a VPS).
> Se validado, ele configura o hostname automaticamente dentro do sistema
> operacional, dispensando o comando `hostnamectl`.

_Repita o processo para `kvm4` e `kvm8` ajustando os nomes e domínios
adequados._

## 3. Firewall de Borda (hPanel)

Antes de configurar o firewall interno (UFW), configuramos o **Firewall da
Hostinger** (hPanel) para proteger a rede antes mesmo que o tráfego chegue nas
VPSs.

> **Se você não usa hPanel:** Replique as mesmas regras no firewall de borda do
> seu provedor/hypervisor (conceito idêntico: whitelist + drop final).

A política adotada é **Whitelist**: Bloqueia tudo (Drop) e libera apenas o
necessário.

**Regras Aplicadas (Aplicar em TODAS as VPS):**

| Ação       | Protocolo | Porta   | Origem (Source)                       | Descrição                                  |
| :--------- | :-------- | :------ | :------------------------------------ | :----------------------------------------- |
| **Accept** | TCP       | Any     | `187.108.118.25` (Seu IP)             | Acesso total do admin (SSH, etc)           |
| **Accept** | TCP       | Any     | `89.116...`, `191.101...`, `76.13...` | Comunicação total entre os nós (Swarm TCP) |
| **Accept** | UDP       | `4789`  | `89.116...`, `191.101...`, `76.13...` | Swarm Overlay Network (VXLAN)              |
| **Accept** | UDP       | `7946`  | `89.116...`, `191.101...`, `76.13...` | Swarm Container Network Discovery          |
| **Accept** | ICMP      | Any     | `89.116...`, `191.101...`, `76.13...` | Ping entre os nós                          |
| **Accept** | ICMP      | Any     | `187.108.118.25` (Seu IP)             | Ping do admin                              |
| **Accept** | HTTPS     | `443`   | `Any` (Qualquer)                      | Tráfego Web Seguro (Traefik)               |
| **Accept** | HTTP      | `80`    | `Any` (Qualquer)                      | Tráfego Web (Traefik)                      |
| **Accept** | UDP       | `51820` | IPs dos Nós + Seu IP                  | WireGuard VPN                              |
| **Accept** | TCP       | `8080`  | `187.108.118.25` (Seu IP)             | Traefik Dashboard (Dev/Debug)              |
| **Drop**   | Any       | Any     | `Any`                                 | **Regra Final: Bloqueia todo o resto**     |

> **Nota:** Certifique-se de substituir o IP `187.108.118.25` pelo **SEU IP**
> atual de internet. Os demais IPs devem ser os IPs das suas outras VPSs.

> **Importante:** Essa configuração protege a borda. Ainda assim, configuraremos
> o `UFW` (firewall local) mais adiante para defesa em profundidade e controle
> da VPN.

## 4. Criação do Usuário e Docker

Ainda logado como `root` (via Terminal web ou SSH), vamos criar o usuário de
trabalho e garantir que ele tenha acesso ao Docker e poderes administrativos.

**Execute em TODAS as VPSs (`kvm2`, `kvm4`, `kvm8`):**

```bash
# Defina o nome do seu usuário
export YOUR_USERNAME="luizotavio"

# Cria o usuário com diretório home (-m) e shell bash (-s)
useradd -m -s /bin/bash $YOUR_USERNAME

# Adiciona ao grupo 'sudo' (admin) e 'docker' (para rodar docker sem sudo)
usermod -aG sudo $YOUR_USERNAME
usermod -aG docker $YOUR_USERNAME

# Define a senha do usuário
passwd $YOUR_USERNAME

# Teste o acesso mudando para o novo usuário
su $YOUR_USERNAME
```

> **Verificação:** Após rodar `su $YOUR_USERNAME`, tente rodar `docker ps`. Se
> funcionar sem erro de permissão, o grupo `docker` foi aplicado corretamente.
> Se pedir senha no `sudo`, está correto.

## 5. Configuração SSH (Chaves e Acesso Fácil)

Agora vamos configurar o acesso seguro e prático a partir do **SEU COMPUTADOR**.
Isso elimina a necessidade de digitar senhas e IPs o tempo todo.

**1. Gerar par de chaves SSH (no seu PC):** Se você ainda não tem uma chave
específica para este projeto:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_hostinger -C "luizotavio"
```

**2. Enviar a chave pública para as VPSs:** Isso autoriza sua chave a entrar nos
servidores. Repita para cada IP ou domínio configurado.

```bash
ssh-copy-id -i ~/.ssh/id_hostinger.pub luizotavio@inprod.cloud
ssh-copy-id -i ~/.ssh/id_hostinger.pub luizotavio@otaviomiranda.cloud
ssh-copy-id -i ~/.ssh/id_hostinger.pub luizotavio@myswarm.cloud
```

**3. Criar "apelidos" no `~/.ssh/config` (no seu PC):** Para não precisar
digitar `ssh luizotavio@inprod.cloud` toda hora, vamos criar atalhos
(`ssh kvm2`).

Edite ou crie o arquivo `~/.ssh/config`:

```text
Host kvm2
  IgnoreUnknown AddKeysToAgent,UseKeychain
  AddKeysToAgent yes
  HostName inprod.cloud
  User luizotavio
  Port 22
  IdentityFile ~/.ssh/id_hostinger

Host kvm4
  IgnoreUnknown AddKeysToAgent,UseKeychain
  AddKeysToAgent yes
  HostName otaviomiranda.cloud
  User luizotavio
  Port 22
  IdentityFile ~/.ssh/id_hostinger

Host kvm8
  IgnoreUnknown AddKeysToAgent,UseKeychain
  AddKeysToAgent yes
  HostName myswarm.cloud
  User luizotavio
  Port 22
  IdentityFile ~/.ssh/id_hostinger
```

**Teste:** Agora basta digitar:

```bash
ssh kvm2
```

Se conectar direto, está tudo pronto!

## 6. Sudo sem Senha (Opcional/Demo)

Para agilizar o desenvolvimento e evitar digitar a senha de `sudo` repetidamente
durante o vídeo/configuração, vamos configurar o `NOPASSWD`.

> **⚠️ ALERTA DE SEGURANÇA:** Em ambientes de produção críticos, isso **não é
> recomendado**. Se um atacante ganhar acesso ao seu usuário, ele ganha acesso
> `root` instantaneamente sem barreiras. Faça isso apenas se entender o risco ou
> para ambientes de laboratório/demo.

**Em cada VPS:**

```bash
# Cria/edita um arquivo específico para seu usuário no sudoers.d
sudo visudo -f /etc/sudoers.d/luizotavio
```

Adicione a linha abaixo (trocando `luizotavio` pelo seu usuário):

```text
luizotavio ALL=(ALL) NOPASSWD: ALL
```

Salve e saia. Agora comandos como `sudo apt update` rodarão direto.

## 7. Atualização e Pacotes Básicos

Vamos garantir que o sistema esteja seguro, atualizado e com nossas ferramentas
favoritas instaladas.

**Execute em TODAS as VPSs:**

```bash
# Define seu fuso horário (ajuste conforme necessário)
export TIMEZONE='America/Sao_Paulo'

# Atualiza repositórios e pacotes do sistema
sudo apt update -y
sudo apt upgrade -y

# Instala ferramentas essenciais
# just: task runner (usaremos muito)
# python3/build-essential: para scripts e compilação
# acl: para controle fino de permissões
sudo apt install -y vim curl ca-certificates htop python3 \
python3-dev acl build-essential tree just

# Aplica o fuso horário
sudo timedatectl set-timezone "$TIMEZONE"
```

## 8. Configuração do Git

Como vamos clonar o repositório e talvez fazer ajustes rápidos, configuramos o
Git com nossa identidade.

**Execute em TODAS as VPSs:**

```bash
# Ajuste com seus dados
export GIT_USERNAME="luizomf"
export GIT_EMAIL="luizomf@gmail.com"

git config --global user.name "$GIT_USERNAME"
git config --global user.email "$GIT_EMAIL"

# Padronização de quebra de linha (Linux style - LF)
git config --global core.autocrlf input
git config --global core.eol lf

# Branch padrão moderna
git config --global init.defaultbranch main
```

## 9. Hardening do SSH (Blindando o Acesso)

Agora vamos trancar as portas. Desabilitaremos login por senha e acesso de root,
deixando apenas nossas chaves autorizadas.

> **⚠️ ALERTA CRÍTICO (Tranqueira à vista):** Este passo **DESATIVA** o login
> por senha.
>
> 1. Certifique-se que você já configurou suas chaves SSH (`ssh-copy-id`) no
>    passo anterior e testou o acesso (`ssh kvm2`).
> 2. Se você rodar isso sem ter a chave configurada, **VOCÊ PERDERÁ O ACESSO
>    SSH** e terá que usar o console de emergência da Hostinger para consertar.

**Execute em TODAS as VPSs:**

```bash
# Instala o servidor SSH (geralmente já vem, mas garante)
sudo apt install -y openssh-server

# Cria nosso arquivo de configuração blindada
cat <<-'EOF' | sudo tee "/etc/ssh/sshd_config.d/01_sshd_settings.conf"
###############################################################################
### Start of /etc/ssh/sshd_config.d/01_sshd_settings.conf ######################
###############################################################################

# BLOCK 1: AUTHENTICATION AND ACCESS
PubkeyAuthentication yes            # Apenas chaves
PasswordAuthentication no           # Senhas desativadas (adeus brute-force)
KbdInteractiveAuthentication no     # Sem teclado interativo
ChallengeResponseAuthentication no  # Sem desafio-resposta
PermitRootLogin no                  # Root nunca loga direto
PermitEmptyPasswords no             # Sem senhas vazias
UsePAM yes                          # PAM ativo para sessão (mas auth é só key)
AuthenticationMethods publickey     # Força auth apenas por chave pública

# BLOCK 2: ATTACK SURFACE REDUCTION
PermitUserEnvironment no            # Bloqueia injeção de env
PermitUserRC no                     # Bloqueia scripts rc de usuário
X11Forwarding no                    # Sem interface gráfica remota

# TUNNELING (Ajuste se precisar de túneis)
AllowTcpForwarding no               # Bloqueia túneis TCP
AllowStreamLocalForwarding no       # Bloqueia socket forwarding
AllowAgentForwarding no             # Bloqueia agent forwarding

PermitOpen none                     # Bloqueia forwarding arbitrário
PermitListen none                   # Bloqueia abrir portas remotas
GatewayPorts no                     # Bloqueia gateway ports
PermitTunnel no                     # Bloqueia interfaces tun/tap

# BLOCK 3: PERFORMANCE & TIMEOUTS
MaxAuthTries 4                      # Max 4 tentativas
LoginGraceTime 30                   # 30s para logar ou tchau
ClientAliveInterval 300             # Keepalive a cada 5 min
ClientAliveCountMax 2               # Cai se não responder 2x
PrintMotd no                        # Menos spam no login
UseDNS no                           # Login rápido (sem reverse DNS lookup)

###############################################################################
### End of /etc/ssh/sshd_config.d/01_sshd_settings.conf ########################
###############################################################################
EOF

# Testa a configuração antes de reiniciar (se der erro, NÃO reinicie)
sudo sshd -t

# Reinicia o serviço SSH para aplicar
sudo systemctl restart ssh
```

> **Dica:** Mantenha sua sessão atual aberta. Abra **outro terminal** no seu PC
> e tente conectar (`ssh kvm2`). Se funcionar, sucesso! Se não, você ainda tem a
> sessão aberta para corrigir.

## 10. Fail2Ban (Proteção Brute-Force)

Mesmo sem senhas, logs de tentativas de acesso poluem o sistema e consomem
recursos. O Fail2Ban bloqueia IPs que tentam conectar e falham repetidamente.

> **Nota do Autor:** Sim, eu sei. Estamos rodando Fail2Ban numa máquina que já
> tem DOIS firewalls (Borda + UFW) bloqueando a porta 22 para todo mundo, exceto
> meu IP. Isso se chama "paranoia saudável" (ou exagero mesmo). Se um dia eu
> errar a config do firewall e abrir a porta sem querer, o Fail2Ban estará lá
> rindo e banindo os bots. 😂

**Execute em TODAS as VPSs:**

```bash
# Defina o IP da sua casa/escritório para nunca ser banido (whitelist)
export ADMIN_SSH_CIDR="187.108.118.25/32" # <-- Troque pelo SEU IP
export FAIL2BAN_IGNOREIP="$ADMIN_SSH_CIDR 127.0.0.1/8 ::1"

# Instala o Fail2Ban
sudo apt install fail2ban -y

# Cria a configuração local (jail.local)
cat <<-EOF | sudo tee "/etc/fail2ban/jail.local"
[DEFAULT]

[sshd]
enabled = true
port = ssh
backend = systemd
# IPs ignorados (você e o localhost)
ignoreip = ${FAIL2BAN_IGNOREIP}

# Regras de banimento
maxretry = 5          # 5 tentativas falhas
findtime = 10m        # dentro de 10 minutos
bantime = 1h          # = Banido por 1 hora

# Banimento progressivo para reincidentes
bantime.increment = true
bantime.factor = 2    # Dobra o tempo a cada reincidência
bantime.max = 24h     # Até o máximo de 24h
EOF

# Reinicia o serviço
sudo systemctl restart fail2ban
```

## 11. Firewall Local (UFW)

Defesa em profundidade. Se o firewall da Hostinger falhar ou for desativado, o
UFW garante que ninguém acessa o que não deve.

Aqui fazemos algo especial: **Liberamos o Swarm APENAS na interface VPN
(`wg0`)**. Se alguém bater no IP público tentando falar com o Docker Swarm, será
bloqueado.

**Execute em TODAS as VPSs:**

```bash
# Defina seu IP e a rede da VPN
export ADMIN_SSH_CIDR="187.108.118.25/32"    # <-- Troque pelo SEU IP
export WG_CIDR="10.100.0.0/24"              # Rede interna que usaremos no WireGuard
export WG_INTERFACE="wg0"                   # Interface do WireGuard

sudo apt install -y ufw

# Zera configurações antigas (garante estado limpo)
sudo ufw disable
sudo ufw --force reset

# Política Padrão: Bloqueia tudo que entra, Libera tudo que sai
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Libera SEU acesso SSH e WireGuard (UDP)
# DICA: Se seu IP não é fixo, libere para "Any" no SSH e confie nas chaves + Fail2Ban
# sudo ufw allow ssh comment "SSH Public"
sudo ufw allow from "$ADMIN_SSH_CIDR" to any port 22 proto tcp comment "SSH Admin"
sudo ufw allow from "$ADMIN_SSH_CIDR" to any port 51820 proto udp comment "WireGuard Admin"

# Libera tráfego do SWARM e NFS APENAS na interface da VPN (wg0)
# Ninguém de fora da VPN consegue tocar nesses serviços
sudo ufw allow in on "$WG_INTERFACE" from "$WG_CIDR" to any port 2377 proto tcp comment "Swarm control (wg)"
sudo ufw allow in on "$WG_INTERFACE" from "$WG_CIDR" to any port 7946 proto tcp comment "Swarm gossip (wg)"
sudo ufw allow in on "$WG_INTERFACE" from "$WG_CIDR" to any port 7946 proto udp comment "Swarm gossip (wg)"
sudo ufw allow in on "$WG_INTERFACE" from "$WG_CIDR" to any port 4789 proto udp comment "Swarm vxlan (wg)"
sudo ufw allow in on "$WG_INTERFACE" from "$WG_CIDR" to any port 2049 proto tcp comment "NFSv4 (wg)"
```

**Execute APENAS no nó `kvm8` (Edge/Traefik):**

```bash
# O kvm8 é o único que recebe tráfego web público
sudo ufw allow 80/tcp comment "HTTP public"
sudo ufw allow 443/tcp comment "HTTPS public"
```

**Finalizar e Ativar (EM TODAS):**

```bash
# Se você errou o IP do SSH lá em cima, você vai cair agora.
sudo ufw --force enable
sudo ufw status verbose
```

## 12. Teste Rápido de Firewall (HTTP)

Vamos subir um servidor web temporário em cada máquina para provar que nossas
regras de firewall funcionam.

**Execute em todos os nós (`kvm2`, `kvm4`, `kvm8`):**

```bash
mkdir test
echo "<div style='display: grid; place-items: center; font-size: 5vw; height: 100vh;'>HELLO FROM $(hostname)</div>" > test/index.html
sudo python3 -m http.server -d test 80
```

**Resultado esperado:**

1. Abra `http://myswarm.cloud` (ou IP do `kvm8`) no navegador -> **FUNCIONA**
   (Você vê "HELLO FROM kvm8").
2. Abra `http://inprod.cloud` (ou IP do `kvm2`) -> **FALHA** (Timeout/Recusado).
3. Abra `http://otaviomiranda.cloud` (ou IP do `kvm4`) -> **FALHA**
   (Timeout/Recusado).

**Para parar:** Pressione `Ctrl+C` no terminal e remova a pasta de teste:

```bash
rm -r test
```

## 13. WireGuard (VPN Privada)

Essa é a parte mais trabalhosa, mas essencial. Vamos criar uma rede privada
(`10.100.0.0/24`) onde os nós conversarão de forma segura e criptografada, sem
expor o Swarm na internet pública.

**Execute em cada VPS, AJUSTANDO a variável `WG_IP` para cada uma:**

```bash
# === PASSO 1: Instalação e Geração de Chaves ===

# Instala o WireGuard
sudo apt install -y wireguard

# Defina a interface e o IP DESTA MÁQUINA (Ajuste para cada VPS!)
export WG_INTERFACE="wg0"
export WG_CIDR="24"
# kvm2=10.100.0.2 | kvm4=10.100.0.4 | kvm8=10.100.0.8
export WG_IP="10.100.0.2" # <--- 🚨 TROQUE ISSO EM CADA MÁQUINA

# Configura caminhos
export WG_DIR="/etc/wireguard"
export WG_CONF="$WG_DIR/$WG_INTERFACE.conf"
export WG_PRI="${WG_DIR}/private.key"
export WG_PUB="${WG_DIR}/public.key"

# Gera chaves e arquivo de config se não existirem
if ! sudo test -f "${WG_PRI}"; then
    echo "Gerando chaves..."
    sudo install -d -m 0700 -o root -g root "${WG_DIR}"
    sudo wg genkey | sudo tee "${WG_PRI}" | sudo wg pubkey | sudo tee "${WG_PUB}" > /dev/null
    sudo chmod 600 "${WG_PRI}"
    sudo chmod 644 "${WG_PUB}"
fi

# Lê as chaves para variáveis
WG_PRI_VALUE=$(sudo cat "${WG_PRI}")
WG_PUB_VALUE=$(sudo cat "${WG_PUB}")

echo "Sua Public Key: ${WG_PUB_VALUE}"

# Cria o arquivo de configuração inicial (com Peers comentados)
cat <<-EOF | sudo tee $WG_CONF
[Interface]
Address = $WG_IP/$WG_CIDR
ListenPort = 51820
PrivateKey = ${WG_PRI_VALUE}

# --- LEMBRETE DE PEERS (MAPA) ---
# kvm2 -> 10.100.0.2 (Public IP: 76.13.71.178)
# kvm4 -> 10.100.0.4 (Public IP: 191.101.70.130)
# kvm8 -> 10.100.0.8 (Public IP: 89.116.73.152)

# [Peer]
# PublicKey = <CHAVE_PUBLICA_DO_OUTRO_VPS>
# AllowedIPs = <IP_INTERNO_DO_OUTRO_VPS>/32
# Endpoint = <IP_PUBLICO_DO_OUTRO_VPS>:51820
# PersistentKeepalive = 25
EOF

# Habilita e inicia o serviço
sudo systemctl enable wg-quick@wg0
sudo systemctl restart wg-quick@wg0
sudo wg show
```

### Passo 2: O " troca-troca" de chaves (Manual)

Agora vem a parte manual. Você precisa editar o arquivo
`/etc/wireguard/wg0.conf` em cada máquina e adicionar os blocos `[Peer]` das
**outras duas máquinas**.

Use `sudo wg show` em cada terminal para ver a Public Key de cada um e monte o
quebra-cabeça.

**Exemplo de como deve ficar o arquivo no `kvm2`:**

```ini
[Interface]
Address = 10.100.0.2/24
...

[Peer] # kvm4
PublicKey = <PUBLIC_KEY_DO_KVM4>
AllowedIPs = 10.100.0.4/32
Endpoint = 191.101.70.130:51820
PersistentKeepalive = 25

[Peer] # kvm8
PublicKey = <PUBLIC_KEY_DO_KVM8>
AllowedIPs = 10.100.0.8/32
Endpoint = 89.116.73.152:51820
PersistentKeepalive = 25
```

Após editar, aplique as mudanças:

```bash
sudo systemctl restart wg-quick@wg0
sudo wg show
```

**Teste de Ping (Fundamental):** Do `kvm2`, tente pingar os IPs internos dos
outros:

```bash
ping 10.100.0.4
ping 10.100.0.8
```

Se pingar, parabéns! Sua rede privada criptografada está de pé.

## 14. Configuração do NFS (Storage Compartilhado)

Como estamos em um cluster, precisamos de um local comum para que arquivos (como
os jobs do webhook) sejam vistos por todos, independente de onde o serviço
esteja rodando. O `kvm8` será nosso servidor de arquivos.

### Servidor NFS (Apenas no kvm8)

Configuramos o servidor com permissões restritas (UID/GID 1011) para alinhar com
o usuário que rodará dentro dos containers.

```bash
# Instala o servidor
sudo apt-get install -y nfs-kernel-server

# Cria os diretórios
sudo mkdir -p /srv/nfs/swarm_data/webhook_jobs

# === Permissões e Grupos ===
# Cria grupo 'app' com GID 1011 (mesmo ID usado no container da API)
sudo groupadd -g 1011 app || true
# Adiciona seu usuário ao grupo para você poder gerenciar arquivos
sudo usermod -aG app "$USER" || true

# Aplica permissões
sudo chown -R root:app /srv/nfs/swarm_data/webhook_jobs
sudo chmod 2770 /srv/nfs/swarm_data/webhook_jobs
# ACLs para garantir que novos arquivos herdem as permissões
sudo setfacl -R -m g:app:rwx /srv/nfs/swarm_data/webhook_jobs
sudo setfacl -R -m d:g:app:rwx /srv/nfs/swarm_data/webhook_jobs

# Aplica mudanças de grupo na sessão atual
newgrp app

# === Exportação (Compartilhamento) ===
# Define a regra de exportação para a rede VPN (10.100.0.0/24)
# all_squash + anonuid=1011: Força tudo a ser escrito como usuário 'app'
EXPORT_LINE="/srv/nfs/swarm_data 10.100.0.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1011,anongid=1011,fsid=0)"
sudo grep -qF "$EXPORT_LINE" /etc/exports || echo "$EXPORT_LINE" | sudo tee -a /etc/exports

# Aplica e verifica
sudo exportfs -ra
sudo exportfs -v

sudo mkdir -p /mnt/nfs
#Monta o volume LOCALMENTE (Sem passar pela rede)
mount --bind /srv/nfs /mnt/nfs

#Configura a montagem automática no Boot
echo "/srv/nfs /mnt/nfs none bind 0 0" | sudo tee -a /etc/fstab
```

### Montagem do volume compartilhado
Todos os nós precisam montar a pasta `/mnt/nfs` para garantir que o caminho seja
idêntico em todo o cluster. 
- Todos os **nodes clients** a monta via rede (passando pela VPN)
- O **server node** a monta via bind mount (Acesso local ao diretório, e o tráfego não passa na rede). 

### Clientes NFS (kvm2, kvm4)

```bash
# Instala o cliente
sudo apt-get install -y nfs-common
sudo mkdir -p /mnt/nfs

# Cria o grupo localmente para alinhar permissões (opcional, mas recomendado)
sudo groupadd -g 1011 app || true
sudo usermod -aG app "$USER" || true

# Configura a montagem automática no Boot (/etc/fstab)
# Destaque para x-systemd.requires=wg-quick@wg0.service:
# Só tenta montar DEPOIS que a VPN subir.
echo "10.100.0.8:/ /mnt/nfs nfs4 rw,vers=4.2,_netdev,noatime,nofail,x-systemd.automount,x-systemd.idle-timeout=600,x-systemd.device-timeout=10s,x-systemd.mount-timeout=30s,x-systemd.requires=wg-quick@wg0.service 0 0" | sudo tee -a /etc/fstab

sudo systemctl daemon-reload

# Monta tudo
sudo mount -a

# Verifica se montou
findmnt /mnt/nfs
```

### Validação Final de Permissões

Teste se conseguimos escrever na pasta compartilhada (como se fossemos o app).
Execute em qualquer nó:

```bash
# Tenta criar um arquivo de teste
mktemp -p /mnt/nfs/webhook_jobs perm_test.XXXXXX

# Verifica permissões
ls -ld /mnt/nfs/webhook_jobs

# Limpa sujeira
sudo rm -f /mnt/nfs/webhook_jobs/perm_test.*
```

## 15. Deploy Keys (Acesso ao Git)

Precisamos baixar o código do projeto nas VPSs. Para não usar sua senha pessoal,
usaremos "Deploy Keys" (chaves SSH específicas para leitura do repositório).

**Execute em cada VPS (`kvm2`, `kvm4`, `kvm8`):**

1.  **Gere a chave de acesso:**

    ```bash
    ssh-keygen -t ed25519
    # Pressione ENTER para todas as perguntas (local padrão, sem senha)
    ```

2.  **Pegue a chave pública:**

    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```

    _Copie o conteúdo que aparece (começa com `ssh-ed25519 ...`)._

3.  **Adicione no GitHub:**
    - Vá no seu repositório -> **Settings** -> **Deploy Keys**.
    - Clique em **Add deploy key**.
    - **Title:** `kvm2` (ou o nome do VPS).
    - **Key:** Cole a chave pública.
    - **Allow write access:** Deixe desmarcado (somente leitura é mais seguro).
    - Clique em **Add key**.

_Repita para todas as 3 máquinas._

## 16. Clone do Repositório

Agora trazemos o código para dentro dos servidores. Usaremos
`/opt/dockerswarmp1` como padrão.

**Execute em TODAS as VPSs:**

```bash
# Cria o diretório e ajusta permissão para seu usuário
sudo mkdir -p /opt/dockerswarmp1
sudo chown -R "$USER:$USER" /opt/dockerswarmp1

# Clona o repositório (use a URL SSH para usar a Deploy Key)
# 🚨 IMPORTANTE: Use o SEU repositório aqui
git clone git@github.com:luizomf/dockerswarmp1.git /opt/dockerswarmp1
```

> **Verificação:** Rode `ls /opt/dockerswarmp1` e veja se os arquivos
> apareceram.

## 17. Inicializando o Swarm

Agora unimos as máquinas em um cluster. Usaremos os IPs da VPN (`10.100.0.x`)
para que o tráfego de gestão do Swarm passe dentro do túnel criptografado.

**1. Iniciar no Líder (Execute no `kvm8`):**

```bash
# --advertise-addr garante que o Swarm use o IP da VPN
docker swarm init --advertise-addr 10.100.0.8
```

_Copie o comando `docker swarm join ...` que vai aparecer na tela._

**2. Adicionar os Nós (Execute no `kvm2` e `kvm4`):** Cole o comando que você
copiou. Deve ser parecido com:

```bash
docker swarm join --token SWMTKN-1-xxxxx 10.100.0.8:2377
```

**3. Promover a Gerentes (Execute no `kvm8`):** Por padrão, os novos nós entram
como "Workers". Vamos promovê-los a "Managers" para ter alta disponibilidade (se
o kvm8 cair, outro assume a gestão).

```bash
docker node promote kvm2 kvm4
```

**4. Validar o Cluster (Execute no `kvm8`):**

```bash
docker node ls
```

O resultado esperado é ver 3 nós, todos com `MANAGER STATUS` preenchido
(Leader + Reachable).

```text
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
ikfyqsoeqxybu...              kvm2       Ready     Active         Reachable
9dxwy9io1zuan...              kvm4       Ready     Active         Reachable
jo48gk4elvo4l... *            kvm8       Ready     Active         Leader
```

## 18. Variáveis de Ambiente (.env)

O `just` (nosso task runner) precisa saber qual domínio usar e outros detalhes.
Vamos configurar o `.env` apenas no nó que usaremos para fazer o deploy
(geralmente o `kvm8`, mas como todos são managers, pode ser em qualquer um).

**Execute no `kvm8`:**

```bash
cd /opt/dockerswarmp1
cp .env.example .env
vim .env
```

**O que você DEVE mudar:**

1. `CURRENT_ENV`: mude para `production` (isso ativa o TLS real no Traefik).
2. `EMAIL`: seu e-mail para o Let's Encrypt.
3. `APP_DOMAIN`: o domínio que você apontou para o `kvm8` (ex: `myswarm.cloud`
   ou `app.myswarm.cloud`).
4. `GITHUB_WEBHOOK_SECRET`: Gere um hash
   (`python3 -c "import secrets; print(secrets.token_hex(32))"`) e guarde.
   Usaremos o mesmo no GitHub.
5. `POSTGRES_PASSWORD`: Use `ANY_VALUE` aqui, pois a senha real virá de um
   **Docker Secret** no próximo passo.

**Exemplo:**

```bash
CURRENT_ENV=production
EMAIL="seu@email.com"
APP_DOMAIN="myswarm.cloud"
GITHUB_WEBHOOK_SECRET="f6a7d8..."
```

## 19. Redes e Labels (Arquitetura)

Antes do deploy, precisamos preparar o terreno:

1.  **Labels:** Marcar o `kvm8` para que o Swarm saiba que este é o nó
    "Especial" (onde ficarão o Banco de Dados e o Traefik).
2.  **Redes:** Criar as redes Overlay (que funcionam sobre o WireGuard).

**Execute no `kvm8`:**

```bash
cd /opt/dockerswarmp1

# === LABEL ===
# Marca o nó atual como 'kvm8'.
# Nossos serviços 'postgres' e 'traefik' têm uma regra: "Rode apenas onde role == kvm8"
docker node update --label-add role=kvm8 kvm8

# === REDES ===
# 'public': Rede onde o Traefik escuta e roteia o tráfego externo.
docker network create --driver=overlay --attachable public

# 'internal': Rede fechada onde API e Banco conversam. Sem acesso externo direto.
docker network create --driver=overlay --attachable --internal internal
```

## 20. Autenticação no GHCR (Imagens Privadas)

Nossas imagens Docker estão hospedadas no GitHub Container Registry (GHCR) e são
privadas. Precisamos autenticar o cluster para baixá-las.

**1. Gerar Token no GitHub:**

- Vá em **Settings** > **Developer Settings** > **Personal access tokens** >
  **Tokens (classic)**.
- Clique em **Generate new token (classic)**.
- Dê um nome (ex: `swarm-pull`).
- Marque APENAS o escopo `read:packages`.
- Copie o token gerado (começa com `ghp_...`).

**2. Login no Docker (Execute no `kvm8`):**

```bash
export GITHUB_USER="SEU_USUARIO_GITHUB"
export GHCR_PAT="COLE_SEU_TOKEN_AQUI"

echo "$GHCR_PAT" | docker login ghcr.io -u "$GITHUB_USER" --password-stdin
```

_Se aparecer "Login Succeeded", estamos prontos._

> **Nota:** Como faremos o deploy usando `--with-registry-auth`, basta logar no
> Manager (`kvm8`) que ele repassa as credenciais para os Workers baixarem as
> imagens.

## 21. Secrets (Segurança Máxima)

Em vez de deixar senhas em arquivos de texto (`.env`), usamos o **Docker
Secrets**. O Swarm encripta esses dados e só entrega na memória do container que
precisa deles.

**Execute no `kvm8`:**

```bash
# Gere uma senha forte para o Webhook (Python one-liner)
python3 -c "import secrets; print(secrets.token_hex(32))"
# COPIE O RESULTADO!

# Cria o secret do Webhook
# (Substitua VALOR_COPIADO pelo hash que você gerou)
printf '%s' "VALOR_COPIADO" | docker secret create github_webhook_secret -

# Cria o secret do Banco de Dados
# (Pode gerar outro hash ou escolher uma senha forte sua)
printf '%s' "MINHA_SENHA_ULTRA_FORTE_DO_BANCO" | docker secret create postgres_password -

# Valide se foram criados
docker secret ls
```

### 🔐 Configuração no GitHub (Actions)

Para que o GitHub consiga "falar" com nosso Webhook com segurança, precisamos
cadastrar esse mesmo segredo lá.

1. Vá no seu repositório -> **Settings** -> **Secrets and variables** ->
   **Actions**.
2. Clique em **New repository secret**.
3. Crie dois secrets:
   - **Nome:** `DEPLOY_WEBHOOK_SECRET`
     - **Valor:** (O mesmo hash que você usou no comando `github_webhook_secret`
       acima)
   - **Nome:** `DEPLOY_WEBHOOK_URL`
     - **Valor:** `https://myswarm.cloud/api/webhook/github` (Ajuste para
       **SEU** domínio do `kvm8`)

> **Importante:** Salve esses valores em um gerenciador de senhas
> (Bitwarden/1Password). Recuperar secrets de dentro do Swarm depois de criados
> dá trabalho.

## 22. Instalar o Watcher (Deploy Automático)

Para que o deploy aconteça automaticamente quando o GitHub nos avisar, usamos um
pequeno script ("watcher") que fica olhando para a pasta do NFS.

**Execute no `kvm8`:**

```bash
cd /opt/dockerswarmp1

# Instala o serviço no systemd
sudo cp scripts/webhook-watcher.service /etc/systemd/system/webhook-watcher.service

# Recarrega o systemd e inicia o serviço
sudo systemctl daemon-reload
sudo systemctl enable --now webhook-watcher

# Verifica se está rodando (deve estar "Active: active (running)")
sudo systemctl status webhook-watcher
```

> **Como testar:** Se você rodar `sudo journalctl -u webhook-watcher -f`, verá o
> log do serviço esperando por arquivos na pasta compartilhada.

## 23. O Grande Deploy

Chegou a hora. Vamos subir a stack pela primeira vez manualmente para garantir
que tudo funciona.

**Execute no `kvm8`:**

```bash
cd /opt/dockerswarmp1

# Carrega as variáveis do .env para o shell atual
# (O 'docker stack deploy' NÃO lê o arquivo .env sozinho!)
set -a; source .env; set +a

# Dispara o deploy
docker stack deploy -d -c docker/stack.yaml dockerswarmp1 --with-registry-auth
```

### Validando se subiu

Agora é monitorar até que todos os serviços estejam "Running".

```bash
# Lista as tarefas rodando
docker stack ps dockerswarmp1 --filter=desired-state=running

# Ver logs específicos (se algo der errado)
docker service logs dockerswarmp1_api -f
docker service logs dockerswarmp1_traefik -f
```

**O que esperar:**

1.  O `postgres` e o `traefik` devem subir no `kvm8`.
2.  A `api` e o `frontend` devem se espalhar pelos nós (`kvm2`, `kvm4`, `kvm8`).
3.  Se tudo estiver verde, acesse seu domínio no navegador!

## 24. Macetes de Observabilidade (Manual)

Cluster distribuído é chato de debugar porque os containers não estão mais "logo
ali". Aqui vão uns truques para não ficar perdido.

### Onde está meu container?

Se você quer entrar na API (`docker exec`), primeiro precisa descobrir em qual
VPS ela caiu.

```bash
# Mostra em qual NÓ (kvm2/4/8) cada réplica está
docker service ps dockerswarmp1_api
```

Depois, acesse o nó via SSH e rode o exec lá:

```bash
ssh kvm2
docker ps | grep api
docker exec -it <CONTAINER_ID> sh
```

### Logs centralizados (Mais ou menos)

Felizmente, você pode ver os logs agregados de TODAS as réplicas a partir do
Manager, sem precisar ir em cada máquina.

```bash
# Vê logs de todas as APIs misturados
docker service logs -f --tail=100 dockerswarmp1_api

# Diferencia quem é quem (adiciona o ID no começo da linha)
docker service logs -f --tail=100 dockerswarmp1_api --no-trunc
```

### Ver a "saúde" visualmente

```bash
# Mostra o status visual de todos os nós e serviços
docker node ls
docker stack ps dockerswarmp1
```

## 25. Manutenção: Rotação de Logs (Essencial)

Por padrão, o Docker guarda logs eternamente. Se sua API for "tagarela", o disco
enche e o servidor trava. Vamos configurar um limite **no Host**.

**Execute em TODOS os nós (`kvm2`, `kvm4`, `kvm8`):**

1.  Crie ou edite `/etc/docker/daemon.json`:

    ```bash
    sudo vim /etc/docker/daemon.json
    ```

2.  Adicione a configuração de rotação (máximo 3 arquivos de 10MB):

    ```json
    {
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

3.  Reinicie o Docker para aplicar:
    ```bash
    sudo systemctl restart docker
    ```

> **Dica Extra (Limpar Logs Agora):** Se você já tem gigas de log e quer zerar
> tudo sem reiniciar containers:
>
> ```bash
> sudo find /var/lib/docker/containers -name "*-json.log" -type f -print -exec truncate -s 0 {} \;
> ```

## 26. Conclusão e Próximos Passos

Parabéns! Você tem um cluster **Docker Swarm** rodando em 3 VPSs, com:

- ✅ **Segurança:** Firewall Borda + UFW + WireGuard + SSH Hardening.
- ✅ **Storage:** NFS com permissões restritas e montagem resiliente.
- ✅ **Rede:** Traefik com TLS automático e rede interna isolada.
- ✅ **Automação:** Webhook para deploy contínuo via GitHub Actions.

**O que ficou de fora (mas vale estudar):**

- **Limites de Recursos:** Definir CPU/RAM limits no `stack.yaml` para evitar
  vizinhos barulhentos.
- **Monitoramento:** Prometheus + Grafana para métricas reais.
- **Backup:** Script para dump do Postgres e rsync da pasta do NFS.

Agora é com você. Divirta-se com seu Swarm! 🚀
