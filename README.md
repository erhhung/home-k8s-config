# Erhhung's Home Kubernetes Cluster Configuration

This project manages the configuration and user files for Erhhung's Kubernetes cluster at home.

## Ansible Vault

Store `ansible_become_pass` in Ansible Vault:

```bash
cd ansible

export ANSIBLE_CONFIG=./ansible.cfg
VAULTFILE="group_vars/all/vault.yml"

ansible-vault create $VAULTFILE
ansible-vault edit   $VAULTFILE
```

The Ansible Vault password is stored in macOS Keychain under item "`Home-K8s`" for account "`ansible-vault`".

## Connections

All managed hosts are running **Ubuntu 24.04** with SSH key from https://github.com/erhhung.keys already authorized.  

Ansible will authenticate as user `erhhung` using private key "`~/.ssh/erhhung.pem`";  
however, all privileged operations using `sudo` will require the password stored in Vault.

## Playbooks

Set the config variable first for the `ansible-playbook` commands below:

```bash
export ANSIBLE_CONFIG=./ansible.cfg
```

1. Install required packages

    ```bash
    ansible-playbook packages.yml
    ```

2. Configure system settings

    2.1. **Host**: host name, time zone, and locale  
    2.2. **Network**: DNS servers and search domains  
    2.3. **Login**: Customize login MOTD messages

    ```bash
    ansible-playbook basics.yml
    ```

3. Set up admin user's home directory

    3.1. **Dot files**: `.bash_aliases`, `.emacs`

    ```bash
    ansible-playbook files.yml
    ```

4. Set up Kubernetes cluster with RKE2

    Installs RKE2 with single control plane node
    and 3 worker nodes.

    ```bash
    ansible-playbook cluster.yml
    ```

Alternatively, **run all 4 playbooks** from the project root folder:

```bash
./play.sh
```
