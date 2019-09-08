Role Ninux Common
=========

[![N|Solid](http://basilicata.ninux.org/images/Logo_Ninux_Basilicata_600-192.png)](http://basilicata.ninux.org)

E' un ruolo per installare, configurare i pacchetti e servizi di base con alcune personalizzazioni.
Il Common configura principalmente:
- porta ssh
- utenti e chiavi ssh
- banner
- configura il profile per gli alias, colori nel terminale

Requirements
------------

## Installazione di Ansible su Debian
Aggiungere la seguente linea al file /etc/apt/sources.list ed eseguire i comandi per installare Ansible
~~~
deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main

$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
$ sudo apt-get update
$ sudo apt-get install ansible
~~~

Role Variables
--------------

- defaults/main.yml vengono definite le seguenti variabili:
  - Porta SSH per non utilizzare quella di default
  - IPv4 forward (abilitazione o disabilitazione del forward in /etc/sysctl.conf
  - gli utenti da aggiungere in sudoers
- vars/main.yml vengono elencati i pacchetti che saranno installati
- task/main.yml viene fatto un Harden al file di configurazione sshd (ATTENZIONE, viene bloccata l'autenticazione con password)


Example Playbook
----------------

Scarica il playbook via Ansible Galaxy
```
ansible-galaxy install mikysal78.ninux_common
```
Un esempio del playbook è questo (hosts puà essere anche impostato su common o solo su locale (vedi il tuo file inventory (hosts):

```Yaml
- hosts: all
  become: "{{ sudo | default('yes') }}"
  roles:
    - mikysal78.ninux_common
  vars:
    # common
    common_ipv4_forward: 1
    common_ssh_port: 2400
    # variabili per ruolo common
    users:
      - name: michele
        authorized:
          - ./keys/michele.pub
```

Nel playbook creare una directory "keys" dove andare ad inserire le chiavi pubbliche degli utenti
Creare un file hosts (inventory) con una sintassi di esempio tipo:
```
[common]
tux.basilicata.nnxx ansible_user=root ansible_port=22 ansible_host=10.27.22.5

[locale]
mikytux ansible_user=root ansible_connection=local
```

Nota: per mia abitudine subito dopo l'installazione del sistema operativo carico subito la mia chiave pubblica sul server nell'utente di root.
Ricordarsi che il ruolo imposta di default la porta ssh sulla 2400, quindi se viene eseguito più volte va modificata nell'inventory (hosts)

License
-------

BSD

Author Information
------------------

- [MikySal78](https://github.com/mikysal78)
- [Nemesisdesign](https://github.com/nemesisdesign)
