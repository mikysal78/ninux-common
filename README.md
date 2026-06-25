# Ansible Role: `mikysal78.ninux_common`

[![N|Solid](http://basilicata.ninux.org/images/Logo_Ninux_Basilicata_600-192.png)](http://basilicata.ninux.org)

Ruolo Ansible per la configurazione di base dei nodi/server della rete
**Ninux**: installa i pacchetti essenziali, mette in sicurezza SSH, crea utenti
e chiavi, personalizza shell/banner e abilita gli aggiornamenti automatici.

L'obiettivo è portare una macchina appena installata a uno stato "pronto e
sicuro" con un solo comando, in modo ripetibile e idempotente.

---

## Indice

- [Cosa configura](#cosa-configura)
- [Compatibilità](#compatibilità)
- [Requisiti](#requisiti)
- [Installazione](#installazione)
- [Variabili del ruolo](#variabili-del-ruolo)
- [Esempio d'uso](#esempio-duso)
- [Avvisi di sicurezza importanti](#avvisi-di-sicurezza-importanti)
- [Note cross-distro](#note-cross-distro)
- [Esecuzioni parziali (tag)](#esecuzioni-parziali-tag)
- [Idempotenza e migrazione](#idempotenza-e-migrazione)
- [Licenza e autori](#licenza-e-autori)

---

## Cosa configura

- **Pacchetti di base**: strumenti di sistema e diagnostica (vedi `vars/main.yml`).
- **Hardening SSH** tramite drop-in in `/etc/ssh/sshd_config.d/`:
  porta personalizzata, login solo a chiave, root solo a chiave, niente
  password, limiti su tentativi/sessioni, `DenyUsers`, ecc.
- **fail2ban** con jail `sshd` (backend systemd).
- **Utenti e chiavi SSH** (chiavi incollate inline nel playbook).
- **sudo** senza password per i gruppi indicati (`/etc/sudoers.d/ninux`).
- **Banner / motd** con il logo Ninux in ASCII.
- **fastfetch** al login, con FQDN e IP locale (IPv4 + IPv6).
- **Shell**: `bash.bashrc` con alias e prompt colorato, `screenrc`, `vimrc`.
- **sysctl**: abilitazione/disabilitazione dell'IPv4 forwarding.
- **unattended-upgrades**: aggiornamenti di sicurezza automatici.
- **hostname** e **/etc/hosts** coerenti con l'inventory.
- **`up2date`**: comando helper per `apt update && apt dist-upgrade`.

---

## Compatibilità

- **Debian 12 (Bookworm)** e **Debian 13 (Trixie)**
- **Ubuntu 22.04 (Jammy)** e **24.04 (Noble)**

Le parti specifiche per distribuzione sono gestite automaticamente:

- **fastfetch** viene preso dai repository dove disponibile
  (Debian ≥ 13, Ubuntu ≥ 25.04) e altrove dal pacchetto `.deb` ufficiale di
  GitHub (architettura rilevata automaticamente: amd64, aarch64, armv7l, …).
- **SSH socket activation**: su Ubuntu (≥ 22.10) sshd è attivato via
  `ssh.socket`; il ruolo passa al servizio classico così la porta del drop-in
  viene applicata. Su Debian il passaggio viene saltato in automatico.

> Requisito comune: `systemd`. In container minimali senza systemd/D-Bus alcuni
> task (restart servizi) potrebbero non funzionare; sui server reali è ok.

---

## Requisiti

- **Ansible** ≥ 2.14 sul nodo di controllo.
- **Nessuna collection esterna**: il ruolo usa solo moduli `ansible.builtin`.

Sul target è sufficiente un'installazione standard di Debian/Ubuntu con accesso
SSH come root (o un utente con `become`).

---

## Installazione

Tramite Ansible Galaxy:

```bash
ansible-galaxy install mikysal78.ninux_common
```

Oppure con un file `requirements.yml`:

```yaml
---
roles:
  - name: mikysal78.ninux_common
    src: https://github.com/mikysal78/ninux-common.git
    version: master
```

```bash
ansible-galaxy install -r requirements.yml -p roles
```

---

## Variabili del ruolo

### `defaults/main.yml` (sovrascrivibili)

| Variabile | Default | Descrizione |
|---|---|---|
| `common_ssh_port` | `2400` | Porta su cui mettere sshd. |
| `common_ipv4_forward` | `0` | `1` per abilitare l'IPv4 forwarding. |
| `common_permit_root_login` | `"prohibit-password"` | Root via SSH: `prohibit-password` (solo chiave), `no` (vietato), `yes` (anche password — sconsigliato). |
| `common_ssh_max_auth_tries` | `3` | `MaxAuthTries`. |
| `common_ssh_max_sessions` | `4` | `MaxSessions`. |
| `common_ssh_deny_users` | `[ubuntu]` | Utenti a cui vietare SSH (`DenyUsers`). |
| `base_svr_groups` | _(lista)_ | Gruppi abilitati a `sudo` NOPASSWD. |

### `vars/main.yml`

- `packages`: elenco dei pacchetti installati (nomi verificati su Debian 13).
- `obsolete_packages`: pacchetti da rimuovere se presenti (es. `screenfetch`).

> Le variabili in `vars/` hanno priorità alta: per modificarle conviene agire
> sui `packages` da inventory/playbook con lo stesso nome, oppure forkare.

### Variabile `users` (da definire nel playbook)

Lista di utenti con le rispettive chiavi pubbliche **inline**:

```yaml
users:
  - name: michele
    authorized:
      - "ssh-ed25519 AAAA... commento"
      - "ssh-ed25519 AAAA... altra-chiave"
```

Ogni utente viene creato con shell `/bin/bash` e aggiunto al gruppo `sudo`.

---

## Esempio d'uso

`server.yaml`:

```yaml
---
- hosts: common
  become: "{{ become | default('yes') }}"
  roles:
    - mikysal78.ninux_common
  vars:
    common_ipv4_forward: 1
    common_ssh_port: 2400
    users:
      - name: root
        authorized:
          - "ssh-ed25519 AAAA... michele@host"
      - name: michele
        authorized:
          - "ssh-ed25519 AAAA... michele@host"
```

Inventory `hosts`:

```ini
[all:vars]
ansible_python_interpreter=/usr/bin/python3

[common]
# prima esecuzione: la macchina è ancora sulla porta 22
tux.basilicata.nnxx ansible_host=10.27.22.5 ansible_user=root ansible_port=22

[locale]
mikytux ansible_connection=local
```

Esecuzione:

```bash
ansible-playbook -i hosts server.yaml
```

---

## Avvisi di sicurezza importanti

1. **Carica la tua chiave pubblica su `root` PRIMA di lanciare il ruolo.**
   Il ruolo imposta `PasswordAuthentication no`: senza una chiave valida già
   presente rischi di restare chiuso fuori.
2. **La porta SSH diventa 2400.** Alle esecuzioni successive aggiorna
   `ansible_port=2400` nell'inventory (alla prima run sei ancora sulla 22).
3. **`PermitRootLogin` di default è `prohibit-password`**: root può entrare
   solo a chiave. Imposta `no` per vietarlo del tutto.
4. **`DenyUsers ubuntu`**: su cloud image Ubuntu l'utente di default è `ubuntu`.
   Se amministri quella macchina con quell'utente, rimuovilo da
   `common_ssh_deny_users` prima di lanciare.
5. Il riavvio di sshd **non interrompe** la sessione SSH in corso: puoi lanciare
   il ruolo da remoto in sicurezza (ma tieni aperta una seconda sessione finché
   non verifichi il login sulla nuova porta).

---

## Note cross-distro

- **fastfetch su Ubuntu LTS (22.04/24.04)** non è nei repo: il ruolo lo installa
  dal `.deb` di GitHub. Serve accesso a `github.com` dal target.
- **`ssh.socket` (Ubuntu)**: viene disattivato in favore di `ssh.service` così
  la porta personalizzata ha effetto.
- **`nyancat`** sta nel componente *universe* di Ubuntu (abilitato di default
  su Ubuntu Server). Su immagini minimali senza universe, abilitalo prima.

---

## Esecuzioni parziali (tag)

Puoi eseguire solo una parte del ruolo con `--tags`:

| Tag | Ambito |
|---|---|
| `apt` | cache, pacchetti base, fastfetch, upgrade |
| `users` | utenti, chiavi SSH, sudoers |
| `ssh` | hardening sshd, drop-in, ssh.socket |
| `security` | fail2ban, intervalli unattended-upgrade |
| `rc` | bashrc, screenrc, vimrc, config fastfetch |
| `update` | helper `up2date` |
| `extra` | banner/motd |
| `hostname` | hostname e `/etc/hosts` |
| `ipv4forward` | sysctl IPv4 forwarding |

Esempio (solo hardening SSH):

```bash
ansible-playbook -i hosts server.yaml --tags ssh
```

---

## Idempotenza e migrazione

Il ruolo è idempotente: rilanciarlo non causa modifiche se nulla è cambiato.
Include inoltre alcuni passi di **migrazione** dai vecchi setup:

- rimuove le righe di hardening scritte in passato direttamente in
  `/etc/ssh/sshd_config` (ora gestite dal drop-in);
- rimuove il vecchio hack del symlink `sysctl.conf → custom.conf`;
- normalizza i `~/.bashrc` esistenti che chiamavano `screenfetch`;
- rimuove il pacchetto `screenfetch`.

---

## Licenza e autori

- **Licenza**: BSD-3-Clause
- **Autori**: [MikySal78](https://github.com/mikysal78),
  [Nemesisdesign](https://github.com/nemesisdesign)
