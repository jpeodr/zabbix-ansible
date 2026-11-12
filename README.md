# Ansible - Zabbix Agent 2

Instala e configura o **Zabbix Agent 2** em vários servidores **Linux** de uma vez, a partir de uma lista de servidores.

O que o playbook faz:

- **Debian/Ubuntu e RHEL/CentOS/Rocky/Alma/Oracle:** adiciona o repositório oficial, instala o `zabbix-agent2`, configura `Server`, `ServerActive`, `Hostname` e `ListenPort`, e inicia o serviço.
- **Opcional:** abre a porta `10050` no firewall (`firewalld` ou `ufw`).

---

## 1. Requisitos

- Um **controlador Linux, macOS ou WSL**. O Ansible **não roda nativamente no Windows**.
- Acesso **SSH com sudo** nos servidores.
- Um **Zabbix Server já funcionando**.
- A versão do agent deve ser compatível com a do server. Normalmente use a **mesma série**: server 7.0 -> agent 7.0, server 6.0 -> agent 6.0, e assim por diante.

---

## 2. Instalar o Ansible (se ainda não tiver)

No WSL/Ubuntu:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible
```

> Rode `source ~/ansible-venv/bin/activate` novamente em cada sessão nova do terminal.

---

## 3. Baixar o projeto

```bash
git clone https://github.com/jpeodr/zabbix-ansible.git
cd zabbix-ansible
```

---

## 4. Colocar os servidores na lista

Abra o arquivo **`inventory.ini`** e descomente/adicione os servidores:

```ini
[servers]
srv-linux-01 ansible_host=10.0.0.11
srv-linux-02 ansible_host=10.0.0.12
srv-linux-03 ansible_host=10.0.0.13
```

O nome antes do `ansible_host` será usado como **Hostname** no Zabbix.

Se o nome no Zabbix for diferente, informe na própria linha:

```ini
srv01 ansible_host=10.0.0.11 zabbix_agent2_hostname=NOME-NO-ZABBIX
```

Também preencha o usuário e a senha na seção `[servers:vars]` do mesmo arquivo.

---

## 5. Configurar o Zabbix Server

Abra **`group_vars/all.yml`** e preencha:

```yaml
zabbix_agent2_server: "10.0.0.5"
zabbix_agent2_branch: "7.0"
```

- `zabbix_agent2_server`: IP ou DNS do **Zabbix Server** (ou proxy).
- `zabbix_agent2_branch`: série do Zabbix. Use a mesma do seu server.

---

## 6. Testar a conexão

```bash
ansible servers -m ping
```

Se todos responderem `pong`, está tudo certo.

---

## 7. Rodar a instalação

Tudo:

```bash
ansible-playbook site.yml
```

Um servidor específico:

```bash
ansible-playbook site.yml --limit srv-linux-01
```

Simular sem alterar (bom para conferir):

```bash
ansible-playbook site.yml --check
```

---

## 8. Conferir se subiu

No servidor:

```bash
systemctl status zabbix-agent2
```

No Zabbix Server, testando de fora:

```bash
zabbix_get -s 10.0.0.11 -k agent.ping
```

Se retornar `1`, o agent está respondendo.

---

## 9. Cadastrar no Zabbix

1. No frontend do Zabbix, crie o host com **o mesmo nome** configurado no agent.
2. Vincule o template **Linux by Zabbix agent**.
3. Para checks passivos, o server precisa alcançar a porta **10050** do host.
4. Para checks ativos, o agent precisa alcançar a porta **10051** do server.

O playbook já configura os dois modos (`Server` e `ServerActive`).

---

## Variáveis principais

| Variável | Padrão | Para que serve |
|---|---|---|
| `zabbix_agent2_server` | vazio | IP/DNS do Zabbix Server ou proxy. **Obrigatório.** |
| `zabbix_agent2_serveractive` | igual ao `server` | Endereço para checks ativos. |
| `zabbix_agent2_hostname` | nome do inventário | Nome do host no Zabbix. |
| `zabbix_agent2_branch` | `7.0` | Série do Zabbix (6.0, 7.0, 7.4, 8.0...). |
| `zabbix_agent2_listenport` | `10050` | Porta de escuta do agent. |
| `zabbix_agent2_hostmetadata` | vazio | Metadados para auto-registro (opcional). |
| `zabbix_agent2_manage_firewall` | `false` | Abre a porta 10050 no firewall. |
| `zabbix_agent2_package_state` | `present` | `latest` atualiza o pacote. |
| `zabbix_agent2_extra_packages` | `[]` | Plugins extras do Agent 2. |

---

## Problemas comuns

**`Missing sudo password`**
Configure `ansible_become_password` no `inventory.ini` ou deixe o usuário com sudo sem senha.

**`UNREACHABLE` / timeout**
Confira o IP, a chave SSH, o usuário e o firewall na porta `22`.

**Agent instalado, mas sem dados no Zabbix**
O `Hostname` do agent precisa ser **igual** ao nome do host no frontend. Corrija no inventário ou use `zabbix_agent2_hostname`.

**Checks passivos falham**
A porta `10050` está fechada. Use `zabbix_agent2_manage_firewall: true` ou abra manualmente.

**Checks ativos falham**
O agent não está alcançando o server na porta `10051`. Confira `ServerActive` e a rede.

**Já existe Zabbix Agent 1 instalado**
O Agent 1 e o Agent 2 usam a mesma porta `10050`. Desinstale o antigo antes, ou altere a porta do novo.

**Sistema não suportado**
O role cobre Debian 11/12/13, Ubuntu 20.04/22.04/24.04/26.04, RHEL 7/8/9/10 e derivados (Rocky, Alma, Oracle, CentOS).

---

## Estrutura do projeto

```text
.
├── ansible.cfg
├── inventory.ini                 # lista de servidores
├── group_vars/all.yml            # configuração do Zabbix
├── site.yml                      # playbook principal
└── roles/zabbix_agent2/          # role que faz a instalação
```

---

## Atualizar o agent depois

Mude `zabbix_agent2_package_state` para `latest` e rode:

```bash
ansible-playbook site.yml
```

O playbook é idempotente: pode rodar quantas vezes quiser.
