# ssh_auto

Automação de distribuição de chaves SSH via Ansible para servidores na Digital Ocean.  
Chega de adicionar e remover chaves manualmente — controle de acesso por equipe, com um único comando.

---

## O problema

Na Digital Ocean, o gerenciamento de acesso SSH era feito **na mão**: entrou alguém no time, adiciona a chave em cada servidor. Saiu alguém, remove de todos. Com múltiplos servidores e equipes diferentes, isso virava bagunça rápido — chave esquecida, acesso sobrando, sem rastreabilidade.

## A solução

Um role Ansible que gerencia o `authorized_keys` de cada servidor com base em **grupos de acesso** definidos em variáveis. Basta declarar quais grupos têm acesso a qual máquina no inventory e rodar o playbook.

---

## Tecnologias

| Ferramenta | Por quê |
|---|---|
| **Ansible** | Idempotente, sem agente, fácil de versionar |
| **blockinfile** | Gerencia blocos nomeados no `authorized_keys` sem sobrescrever o arquivo inteiro |
| **YAML inventory** | Legível, versionável no Git, fácil de revisar em PR |

---

## Estrutura do projeto

```
ssh_auto/
├── inventory/
│   └── production.yml        # Hosts e grupos permitidos por máquina
├── playbooks/
│   └── send_ssh.yml          # Playbook principal
├── roles/
│   └── send_ssh/
│       ├── tasks/
│       │   └── main.yml      # Lógica de distribuição das chaves
│       └── vars/
│           └── main.yml      # ssh_groups e ssh_users (chaves públicas)
└── ansible.cfg
```

---

## Configuração

### 1. Defina os grupos e chaves em `roles/send_ssh/vars/main.yml`

```yaml
ssh_groups:
  frontend:
    - frontend_user1
    - frontend_user2
  backend:
    - backend_user1
    - backend_user2
  devops:
    - devops_user1
  access:
    - access_user1

ssh_users:
  frontend_user1: "ssh-ed25519 AAAA... user1@company"
  frontend_user2: "ssh-ed25519 AAAA... user2@company"
  backend_user1:  "ssh-rsa AAAA... user3@company"
  # ...
```

### 2. Configure o inventory com os grupos permitidos por host

```yaml
# inventory/production.yml
all:
  hosts:
    web-01:
      ansible_host: 123.456.789.0
      ansible_user: root
      ssh_allowed_groups:
        - frontend
        - devops

    api-01:
      ansible_host: 123.456.789.1
      ansible_user: root
      ssh_allowed_groups:
        - backend
        - devops
```

### 3. Rode o playbook

```bash
ansible-playbook -i inventory/production.yml playbooks/send_ssh.yml
```

---

## Como fica o `authorized_keys` gerado

Cada grupo recebe um bloco nomeado — fácil de auditar e seguro para re-execução:

```
# BEGIN ANSIBLE MANAGED SSH KEYS - frontend
ssh-ed25519 AAAA1111 frontend_user1@company
ssh-ed25519 AAAA1112 frontend_user2@company
# END ANSIBLE MANAGED SSH KEYS - frontend

# BEGIN ANSIBLE MANAGED SSH KEYS - devops
ssh-ed25519 AAAA3331 devops_user1@company
ssh-ed25519 AAAA3332 devops_user2@company
# END ANSIBLE MANAGED SSH KEYS - devops
```

> Remover um grupo do inventory e rodar novamente apaga **exatamente aquele bloco**, sem tocar nos outros.

---

## Casos de uso comuns

**Novo membro no time de backend:**  
Adiciona a chave dele em `ssh_users` e o username em `ssh_groups.backend`. Roda o playbook. Pronto.

**Alguém saiu do time:**  
Remove o usuário de `ssh_groups`. Roda o playbook. A chave some de todos os servidores onde o grupo tinha acesso.

**Novo servidor:**  
Adiciona o host no inventory com os grupos permitidos. Roda o playbook.

---

## Requisitos

- Ansible >= 2.9
- Acesso SSH ao(s) host(s) alvo com usuário que tenha permissão de escrita no `authorized_keys`
- Python no host de controle (padrão em qualquer máquina com Ansible)

---

## ansible.cfg recomendado

```ini
[defaults]
host_key_checking = False
```

> Necessário em ambientes com VMs recriadas frequentemente (ex: Vagrant, snapshots).  
> Em produção, prefira adicionar as chaves via `ssh-keyscan` no `known_hosts`.
