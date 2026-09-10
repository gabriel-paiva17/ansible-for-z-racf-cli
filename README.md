# Guia de Início Rápido

Este guia ajudará você a começar rapidamente com o projeto de automação Ansible para criação de usuários RACF.

## ⚡ Início Rápido em 5 Passos

### Pré-requisitos

Certifique-se de ter:
- ✅ Ansible instalado (versão 2.9+)
- ✅ Acesso SSH ao sistema z/OS
- ✅ Permissões RACF adequadas (SPECIAL ou group-SPECIAL)
- ✅ Collection `ibm.ibm_zos_core` instalada

```bash
ansible-galaxy collection install ibm.ibm_zos_core
```

### 1️⃣ Configurar o Inventário

Edite [`inventory/hosts.yml`](../inventory/hosts.yml) com o IP do seu z/OS:

```yaml
zos_system:
  ansible_host: <ip-ou-hostname-do-zos>   # ← Altere aqui
  ansible_user: zosadmn                   # ← Altere aqui
  ansible_python_interpreter: /usr/lpp/IBM/cyp/v3r13/pyz/bin/python3
```

Escolha também o método de autenticação SSH — deixe **um ativo** e o outro comentado:

```yaml
# Opção 1: certificado (chave privada)
# ansible_ssh_private_key_file: /caminho/para/chave.pem  # ← Altere aqui

# Opção 2: senha RACF (via vault)
ansible_ssh_pass: "{{ racf_ssh_password }}"
```

### 2️⃣ Configurar as Variáveis do Usuário

Edite [`vars/user_config.yml`](../vars/user_config.yml):

```yaml
racf_user:
  name: NEWUSER1              # ← Nome do usuário (máx 8 chars)
  display_name: "New User 1"  # ← Nome completo (máx 20 chars)
  owner: ADMIN                # ← Proprietário do perfil RACF
```

> A senha do novo usuário **não fica aqui** — ela está no vault (ver passo 3).

### 3️⃣ Configurar o Ansible Vault

Todas as variáveis secretas ficam em [`vars/secrets.yml`](../vars/secrets.yml).

Se necessário, crie esse arquivo com:

```bash
# Se necessário, crie esse arquivo com:
touch vars/secrets.yml
```

```yaml
# Senha RACF do usuário que conecta via SSH (usado se escolher Opção 2 no inventário)
racf_ssh_password: "CHANGE_ME"

# Senha RACF do novo usuário a ser criado pelo playbook
racf_new_user_password: "CHANGE_ME"
```

#### Primeira vez — criptografar

```bash
# 1. Edite vars/secrets.yml e substitua os CHANGE_ME pelos valores reais
# 2. Criptografe:
ansible-vault encrypt vars/secrets.yml
```

#### Operações comuns no vault

```bash
# Ver o conteúdo descriptografado
ansible-vault view vars/secrets.yml

# Editar diretamente (sem descriptografar em disco)
ansible-vault edit vars/secrets.yml

# Mudar a vault password
ansible-vault rekey vars/secrets.yml
```

### 4️⃣ Teste de Conectividade

> Não está funcionando depois de implementar logon com senha / vault.

```bash
ansible zos_hosts -i inventory/hosts.yml -m ping --ask-vault-pass
```

**Resultado esperado:**
```
zos_system | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> Se estiver usando certificado (Opção 1), o `--ask-vault-pass` ainda é necessário para carregar o `racf_new_user_password`.

### 5️⃣ Criar o Usuário RACF

**Via módulo `zos_user`:**
```bash
ansible-playbook -i inventory/hosts.yml playbooks/create_racf_user.yml --ask-vault-pass
```

**Via comando TSO** (`ADDUSER` — usa `racf_new_user_password` do vault):
```bash
ansible-playbook -i inventory/hosts.yml playbooks/create_racf_user_with_tso.yml --ask-vault-pass
```

#### Alternativa: arquivo de vault password (para pipelines CI/CD)

```bash
# Crie o arquivo com a vault password (nunca commitar — já está no .gitignore)
echo "minha-vault-password" > vars/.vault_pass
chmod 600 vars/.vault_pass

# Execute sem prompt
ansible-playbook -i inventory/hosts.yml playbooks/create_racf_user_with_tso.yml \
  --vault-password-file vars/.vault_pass
```

---

## 🔧 Variáveis de Configuração

| Arquivo | Variável | Descrição |
|---|---|---|
| `inventory/hosts.yml` | `ansible_host` | IP ou hostname do z/OS |
| `inventory/hosts.yml` | `ansible_user` | Usuário SSH de conexão ao z/OS |
| `inventory/hosts.yml` | `ansible_ssh_private_key_file` | Caminho da chave privada (Opção 1) |
| `vars/user_config.yml` | `racf_user.name` | Nome do novo usuário RACF (máx 8 chars) |
| `vars/user_config.yml` | `racf_user.display_name` | Nome completo (máx 20 chars) |
| `vars/user_config.yml` | `racf_user.owner` | Proprietário do perfil RACF |
| `vars/secrets.yml` *(vault)* | `racf_ssh_password` | Senha RACF para autenticação SSH (Opção 2) |
| `vars/secrets.yml` *(vault)* | `racf_new_user_password` | Senha RACF do novo usuário a ser criado |

## 📚 Recursos

- [Documentação IBM z/OS Core Collection](https://ibm.github.io/z_ansible_collections_doc/index.html)
- [Módulo zos_user](https://galaxy.ansible.com/ui/repo/published/ibm/ibm_zos_core/content/module/zos_user/)
- [Módulo zos_tso_command](https://galaxy.ansible.com/ui/repo/published/ibm/ibm_zos_core/content/module/zos_tso_command/)
- [Ansible Vault — documentação oficial](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [README Principal](../README.md)
