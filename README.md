# Projeto 01 – DevOps com Vagrant e Ansible

| | |
|---|---|
| **Instituição** | Instituto Federal da Paraíba – Campus João Pessoa |
| **Departamento** | Departamento de Ensino Superior |
| **Disciplina** | Administração de Sistemas Abertos |
| **Professor** | Leonidas Lima |
| **Período** | 2026.2 |

## Equipe

| Integrante | Matrícula | Login nas VMs |
|---|---|---|
| Guilherme Matias | 20251380012 | `guilherme` |
| Bruno Anderson | 20251380014 | `bruno` |

---

## 1. Descrição do Projeto

O projeto aplica **Infraestrutura como Código (IaC)**: o **Vagrant** provisiona quatro máquinas virtuais Debian 12 no VirtualBox e o **Ansible** configura o sistema operacional e os serviços de cada uma, de forma automatizada e idempotente (executar novamente não altera o que já está correto).

### 1.1 Arquitetura

Rede privada host-only `192.168.56.0/24`. Domínio: `guilherme.bruno.devops`. Gateway da rede: `192.168.56.1` (host).

| VM | Hostname | IP | MAC | RAM | Papel |
|---|---|---|---|---|---|
| `arq` | `arq.guilherme.bruno.devops` | `192.168.56.112` (fixo) | – | 512 MB | Servidor de arquivos: DHCP e DNS autoritativos, LVM e NFS. Possui 3 discos extras de 10 GB |
| `db` | `db.guilherme.bruno.devops` | `192.168.56.114` (DHCP, reserva por MAC) | `08:00:27:24:00:12` | 512 MB | Servidor de banco de dados (MariaDB) + autofs |
| `app` | `app.guilherme.bruno.devops` | `192.168.56.124` (DHCP, reserva por MAC) | `08:00:27:24:00:14` | 512 MB | Servidor de aplicação (Apache2) + autofs |
| `cli` | `cli.guilherme.bruno.devops` | `192.168.56.60` (DHCP, reserva por MAC dentro da faixa `.50`–`.100`) | `08:00:27:24:01:12` | 1024 MB | Cliente: Firefox ESR com X11 Forwarding + autofs |

Regras de endereçamento (derivadas das matrículas): `arq` = `1XX` (dois últimos dígitos da matrícula 1 → `12`); `db` = `1YY` (dois últimos dígitos da matrícula 2 → `14`); `app` = `1XY` (último dígito de cada matrícula → `24`).

```
                     Host (máquina real) 192.168.56.1
                                  |
        ------------------- rede 192.168.56.0/24 -------------------
        |               |                |                 |
   arq .112          db .114          app .124          cli .60
   DHCP + DNS        MariaDB          Apache2           Firefox + X11
   LVM + NFS         autofs           autofs            autofs
        ^                 |                |                 |
        '---- NFS /dados/nfs  <->  /var/nfs (via autofs) ----'
```

### 1.2 Funcionamento

**Vagrant (`Vagrantfile`)**
- Provider VirtualBox, box `debian/bookworm64`, `linked_clone`, verificação de *guest additions* desabilitada e geração de nova chave SSH desativada (`insert_key = false`).
- Gatilho (`trigger`) executado antes de `up`, `reload` e `resume` que **desabilita o servidor DHCP do VirtualBox** da rede `192.168.56.0/24`, para que apenas o DHCP da `arq` responda.
- `db`, `app` e `cli` têm MAC fixo (`080027` + `XY` + sufixo), usado nas reservas do DHCP da `arq`.
- Um *provisioner* de shell em `db`, `app` e `cli` identifica a interface host-only pelo MAC e a configura de forma persistente (`/etc/network/interfaces.d/hostonly`) para obter IP via DHCP da `arq`, com o DNS da `arq` priorizado e rota padrão da host-only com *metric* alto (a saída para a internet continua pela NAT).

**Ansible**
- **Configuração global (todas as VMs):** atualização do sistema (`update`/`dist-upgrade`); `chrony` apontando para `pool.ntp.br` e fuso `America/Recife`; grupo `ifpb`; usuários `guilherme` e `bruno` com chaves SSH ed25519 geradas e instaladas; `sudo` sem senha para o grupo `ifpb` (validado com `visudo`); `nfs-common`; SSH somente com chave pública, `root` bloqueado, `AllowGroups vagrant ifpb` e banner *"Acesso apenas para pessoas com autorização expressa!!!"*.
- **`arq`:** `isc-dhcp-server` (autoritativo, lease padrão 180 s, máximo 3600 s, faixa `.50`–`.100`, gateway `192.168.56.1`, DNS `arq`, reservas por MAC para `db`, `app` e `cli`); `bind9` (aceita a rede interna, *forwarders* `1.1.1.1` e `8.8.8.8`, zonas direta e reversa de `arq`, `db`, `app` e `cli`); LVM (VG `dados` com os 3 discos, LV `ifpb` de 15 GB em `ext4`, montado em `/dados` via `fstab`); NFS (`/dados/nfs` exportado para `192.168.56.0/24`, usuário `nfs-ifpb` sem shell, `all_squash` para `nfs-ifpb`, gravação `sync`).
- **`db`:** `mariadb-server` e `autofs` (monta `/dados/nfs` da `arq` em `/var/nfs`).
- **`app`:** `apache2` com `index.html` próprio (descrição do projeto, nomes e matrículas da equipe) e `autofs`.
- **`cli`:** `firefox-esr`, `xauth`, `X11Forwarding yes` no `sshd` e `autofs`.

### 1.3 Decisões de projeto

- **Fonte única de verdade:** nomes, matrículas, domínio e rede aparecem no `Vagrantfile` e em `group_vars/all.yml`. Se um deles mudar, altere o outro.
- **SSH:** a política fica em `/etc/ssh/sshd_config.d/00-ifpb.conf` (o sshd usa o primeiro valor lido), validada com `sshd -t` antes de aplicar.
- **Reserva do `cli`:** o `cli` obtém o IP via DHCP, por reserva por MAC dentro da faixa dinâmica, o que permite registrar seu nome no DNS.
- **NFS:** `all_squash` mapeia qualquer usuário remoto para `nfs-ifpb`, dono do diretório (`0755`); só ele escreve. `sync` força a gravação imediata em disco.

---

## 2. Estrutura do Repositório

```
.
├── Vagrantfile                  # 4 VMs, discos da arq, trigger do DHCP, rede DHCP das demais VMs
├── ansible.cfg                  # Configuração do Ansible
├── requirements.yml             # Coleções: ansible.posix, community.general, community.crypto
├── verifica-proj1.py            # Script de verificação automática da disciplina
├── inventory/
│   └── hosts                    # Inventário (arq, db, app, cli)
├── group_vars/
│   └── all.yml                  # Variáveis: equipe, rede, IPs, MACs, NFS, SSH
├── files/
│   └── ssh_keys/                # Chaves SSH geradas pelo Ansible (privadas ignoradas pelo git)
├── playbooks/
│   ├── global.yml               # Configuração global (todas as VMs)
│   ├── arq.yml                  # DHCP, DNS, LVM e NFS
│   ├── db.yml                   # MariaDB + autofs
│   ├── app.yml                  # Apache2 + autofs
│   ├── cli.yml                  # Firefox + X11 + autofs
│   ├── etapa3.yml               # Executa arq → db → app → cli em ordem
│   └── site.yml                 # global.yml + etapa3.yml (todas as VMs)
└── roles/
    ├── global/                  # sistema, ntp, usuarios, ssh
    ├── arq/                     # dhcp, dns, lvm, nfs (+ templates)
    ├── db/
    ├── app/                     # templates/index.html.j2
    ├── cli/
    └── autofs_nfs/              # Role compartilhada por db, app e cli
```

---

## 3. Instruções de Execução (do zero)

### 3.1 Pré-requisitos (máquina real)

- VirtualBox 7.x
- Vagrant ≥ 2.4 (em versões 2.2.8 a 2.3.x defina `VAGRANT_EXPERIMENTAL="disks"`)
- Ansible ≥ 2.15 (Linux, macOS ou WSL), Python 3 e a biblioteca `cryptography` (`pip install cryptography`), usada pelo módulo que gera as chaves SSH

### 3.2 Passo a passo

```bash
# 1. Clonar o repositório e instalar as coleções do Ansible
git clone https://github.com/<usuario>/DevOPS-Vagrant-Ansible.git
cd DevOPS-Vagrant-Ansible
ansible-galaxy collection install -r requirements.yml

# 2. Fase 1 – subir a arq e configurá-la (o trigger desabilita o DHCP do VirtualBox)
vagrant up arq
ansible-playbook playbooks/global.yml --limit arq
ansible-playbook playbooks/arq.yml

# 3. Fase 2 – subir db, app e cli: o provisioner de rede obtém o IP no DHCP da arq
vagrant up db app cli

# 4. Fase 3 – configuração global e serviços de todas as VMs
ansible-playbook playbooks/site.yml
```

Por que nessa ordem: `db`, `app` e `cli` dependem do DHCP e do DNS da `arq`; por isso a `arq` é configurada primeiro. Na primeira execução do `global.yml` as chaves SSH dos usuários são geradas em `files/ssh_keys/`.

> Se `db`, `app` ou `cli` tiverem subido antes do DHCP da `arq`, rode `vagrant provision <vm>` (ou `vagrant reload <vm>`) depois da Fase 1.
> Reexecutar os playbooks é seguro: a segunda execução deve terminar com `changed=0` nas tarefas de configuração.

### 3.3 Acesso às VMs

```bash
vagrant ssh arq

ssh -i files/ssh_keys/guilherme guilherme@192.168.56.112
ssh -i files/ssh_keys/bruno bruno@192.168.56.112
```

Para evitar avisos de *host key* ao recriar as VMs, adicione ao `~/.ssh/config` da máquina real:

```
Host *.guilherme.bruno.devops 192.168.56.*
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

### 3.4 Destruir o ambiente

```bash
vagrant destroy -f
```

---

## 4. Checklist de Validação (Critérios de Avaliação)

> Execute os comandos a partir da raiz do repositório, na máquina real, salvo indicação em contrário. Chaves: `files/ssh_keys/guilherme` e `files/ssh_keys/bruno`.

### 4.1 Vagrant (10 pontos)

- [ ] **VMs criadas e em execução**
  ```bash
  vagrant status
  VBoxManage list runningvms
  ```
  Esperado: `arq`, `db`, `app` e `cli` em `running`.

- [ ] **Hardware conforme especificação**
  ```bash
  VBoxManage showvminfo arq.guilherme.bruno.devops | grep -E "Memory size|Description"
  VBoxManage showvminfo cli.guilherme.bruno.devops | grep "Memory size"      # 1024 MB
  vagrant ssh arq -c "lsblk -d -o NAME,SIZE,TYPE | grep disk"                # sda + 3 discos de 10 GB
  ```

- [ ] **DHCP do VirtualBox desabilitado**
  ```bash
  VBoxManage list dhcpservers
  ```
  Esperado: nenhum servidor da rede `192.168.56.x` com `Enabled: Yes` (ou a lista vazia). Durante o `vagrant up` aparece o gatilho **"Desabilitar DHCP"**.
  *(Windows: `"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" list dhcpservers`)*

- [ ] **Quem atribui os IPs é o DHCP da `arq`**
  ```bash
  vagrant ssh arq -c "sudo systemctl is-active isc-dhcp-server"
  vagrant ssh arq -c "grep -E 'default-lease-time|max-lease-time|range|routers|domain-name' /etc/dhcp/dhcpd.conf"
  vagrant ssh arq -c "sudo grep -E '^lease|hardware ethernet' /var/lib/dhcp/dhcpd.leases"
  for vm in db app cli; do vagrant ssh $vm -c "hostname -f; ip -4 -br a"; done
  ```
  Esperado: lease padrão `180`, máximo `3600`, faixa `192.168.56.50 – 192.168.56.100`, gateway `192.168.56.1`, DNS `192.168.56.112`; `db` = `.114`, `app` = `.124`, `cli` = `.60`.

### 4.2 Ansible (80 pontos)

#### a) Acesso por hostname (de `cli`, `db`, `app` e do host)

**A partir de `cli`, `db` e `app`** (o DHCP da `arq` entrega o DNS e o domínio):
```bash
for vm in cli db app; do
  echo "=== $vm ==="
  vagrant ssh $vm -c 'grep nameserver /etc/resolv.conf | head -1; for h in arq db app cli; do ping -c1 -W2 $h.guilherme.bruno.devops >/dev/null && echo "OK    $h" || echo "FALHA $h"; done'
done
```
Esperado: o primeiro `nameserver` é `192.168.56.112` e todos os hosts respondem `OK`.

**A partir do host (máquina real):**
```bash
nslookup arq.guilherme.bruno.devops 192.168.56.112       # 192.168.56.112
nslookup db.guilherme.bruno.devops  192.168.56.112       # 192.168.56.114
nslookup app.guilherme.bruno.devops 192.168.56.112       # 192.168.56.124
nslookup cli.guilherme.bruno.devops 192.168.56.112       # 192.168.56.60
nslookup 124.56.168.192.in-addr.arpa 192.168.56.112      # zona reversa → app.guilherme.bruno.devops

# Para usar os nomes diretamente no host (Linux com systemd-resolved; interface host-only = vboxnet0):
sudo resolvectl dns vboxnet0 192.168.56.112
sudo resolvectl domain vboxnet0 '~guilherme.bruno.devops'
for h in arq db app cli; do ping -c1 -W2 $h.guilherme.bruno.devops >/dev/null && echo "OK    $h" || echo "FALHA $h"; done
ssh -i files/ssh_keys/guilherme guilherme@db.guilherme.bruno.devops hostname -f
```
*(Em Windows/macOS, configure `192.168.56.112` como servidor DNS da interface host-only ou use `nslookup` como acima.)*

#### b) Configurações globais (todas as VMs)

```bash
# Troque H para testar cada VM: 192.168.56.112 (arq), .114 (db), .124 (app), .60 (cli)
H=192.168.56.112
K="-i files/ssh_keys/guilherme"

# Usuários, grupo ifpb e sudo sem senha
ssh $K guilherme@$H 'id; id bruno; getent group ifpb; sudo -n id'

# Fuso horário e NTP (chrony → pool.ntp.br)
ssh $K guilherme@$H 'timedatectl | grep "Time zone"; grep ^pool /etc/chrony/chrony.conf; chronyc sources'

# Cliente NFS
ssh $K guilherme@$H 'dpkg -l nfs-common | tail -1'

# Sistema atualizado
ssh $K guilherme@$H 'sudo apt-get update -qq && apt list --upgradable 2>/dev/null | tail -n +2 | wc -l'   # esperado: 0
```

#### c) Restrições de SSH

```bash
H=192.168.56.112

# Configuração efetiva do sshd
ssh -i files/ssh_keys/guilherme guilherme@$H \
  'sudo sshd -T | grep -Ei "^(permitrootlogin|passwordauthentication|pubkeyauthentication|allowgroups|banner)"'
# Esperado: permitrootlogin no · passwordauthentication no · pubkeyauthentication yes
#           allowgroups vagrant ifpb · banner /etc/issue.net

# Banner de saudação (exibido no início da conexão)
ssh -i files/ssh_keys/bruno bruno@$H true
# Esperado: "Acesso apenas para pessoas com autorização expressa!!!"

# Login com chave pública funciona
ssh -i files/ssh_keys/bruno bruno@$H whoami

# Login por senha é bloqueado
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password guilherme@$H
# Esperado: Permission denied (publickey)

# Login como root é bloqueado
ssh -i files/ssh_keys/guilherme root@$H
# Esperado: Permission denied

# Usuário fora dos grupos vagrant/ifpb é bloqueado, mesmo com chave válida (usuário temporário)
ssh -i files/ssh_keys/guilherme guilherme@$H 'sudo useradd -m intruso'
ssh -i files/ssh_keys/guilherme guilherme@$H \
  'sudo install -d -m700 -o intruso -g intruso /home/intruso/.ssh; sudo tee /home/intruso/.ssh/authorized_keys >/dev/null; sudo chown intruso: /home/intruso/.ssh/authorized_keys' < files/ssh_keys/guilherme.pub
ssh -i files/ssh_keys/guilherme intruso@$H id      # Esperado: Permission denied
ssh -i files/ssh_keys/guilherme guilherme@$H 'sudo userdel -r intruso'
```

#### d) LVM (`arq`)

```bash
vagrant ssh arq -c "sudo pvs"                       # /dev/sdb, /dev/sdc, /dev/sdd → VG dados
vagrant ssh arq -c "sudo vgs dados"
vagrant ssh arq -c "sudo lvs dados/ifpb"            # LV ifpb com 15.00g
vagrant ssh arq -c "lsblk -f"                       # dados-ifpb com ext4 montado em /dados
vagrant ssh arq -c "findmnt /dados; grep dados /etc/fstab; df -hT /dados"
```

#### e) NFS e autofs

```bash
# Servidor (arq)
vagrant ssh arq -c "sudo exportfs -v"               # /dados/nfs 192.168.56.0/24(rw,sync,all_squash,anonuid=...)
vagrant ssh arq -c "getent passwd nfs-ifpb"         # shell /usr/sbin/nologin
vagrant ssh arq -c "ls -ld /dados/nfs"              # dono nfs-ifpb, escrita só para o dono
vagrant ssh arq -c "sudo -u nobody touch /dados/nfs/x"   # Esperado: Permission denied

# Clientes (db, app, cli): montagem automática em /var/nfs via autofs
for vm in db app cli; do vagrant ssh $vm -c "systemctl is-active autofs; ls /var/nfs >/dev/null; findmnt /var/nfs"; done

# Escrita remota (qualquer usuário é mapeado para nfs-ifpb) – a partir do app
ssh -i files/ssh_keys/guilherme guilherme@192.168.56.124 \
  'echo "Dados NFS de guilherme" > /var/nfs/guilherme.txt; ls -l /var/nfs/guilherme.txt'

# Leitura dos mesmos dados a partir do db e do cli, e conferência no servidor
ssh -i files/ssh_keys/guilherme guilherme@192.168.56.114 'cat /var/nfs/guilherme.txt'
ssh -i files/ssh_keys/guilherme guilherme@192.168.56.60  'cat /var/nfs/guilherme.txt'
vagrant ssh arq -c "ls -l /dados/nfs"               # no servidor, o dono do arquivo é nfs-ifpb
```

#### f) Serviços das demais VMs

```bash
# db: MariaDB
vagrant ssh db -c "systemctl is-active mariadb; sudo ss -ntlp | grep mariadb"

# app: Apache2 com index.html do projeto (nomes e matrículas da equipe)
curl -s http://192.168.56.124 | grep -E "Projeto 01|Guilherme|Bruno|2025138"
vagrant ssh cli -c "curl -s http://app.guilherme.bruno.devops | head -n 12"

# cli: Firefox e X11 Forwarding (requer servidor X local; no Windows use VcXsrv ou MobaXterm)
vagrant ssh cli -c "dpkg -l firefox-esr xauth | tail -2; sudo sshd -T | grep x11forwarding"   # x11forwarding yes
ssh -X -i files/ssh_keys/bruno bruno@192.168.56.60 firefox http://app.guilherme.bruno.devops
```

#### g) Verificação automática (script da disciplina)

```bash
cp -r files/ssh_keys keys          # o script espera as chaves em ./keys/<usuario>
python3 verifica-proj1.py guilherme.bruno
```
> Requer a resolução de nomes do host (item a) e o bloco `Host` do `~/.ssh/config` da seção 3.3. O aviso *"Permanently added ... to the list of known hosts"* é inofensivo.

### 4.3 Documentação e organização (10 pontos)

- [x] Integrantes, disciplina e professor – cabeçalho deste README.
- [x] Descrição detalhada do funcionamento – seção 1.
- [x] Instruções de execução – seção 3.
- [x] `Vagrantfile`, inventário (`inventory/hosts`), `ansible.cfg` e playbooks (`playbooks/` + `roles/`) – seção 2.

---

## 5. Solução de Problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| `db`/`app`/`cli` sem IP `192.168.56.x` | Subiram antes do DHCP da `arq` | Conclua a Fase 1 e rode `vagrant provision <vm>` |
| `ansible-playbook` não conecta em `db`/`app`/`cli` | VM sem lease DHCP | Verifique `vagrant ssh <vm> -c "ip -4 -br a"` e o item acima |
| Nomes não resolvem no host | Host não usa o DNS da `arq` | Aplique o `resolvectl` do item 4.2 (a) |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | VMs recriadas | Use o bloco `Host` da seção 3.3 ou `ssh-keygen -R <host>` |
| `ModuleNotFoundError: cryptography` ao gerar chaves | Dependência ausente no controlador | `pip install cryptography` |

---

## 6. Observações de Segurança

- As chaves **privadas** em `files/ssh_keys/` são ignoradas pelo Git (`.gitignore`); apenas as públicas (`*.pub`) são versionadas.
- Os usuários `guilherme` e `bruno` não possuem senha: o acesso é exclusivamente por chave pública.
- O usuário `vagrant` continua com acesso por chave (a chave padrão do Vagrant), exigido pelo grupo `vagrant` em `AllowGroups`.
