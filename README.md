# Détection et analyse d'une compromission sur un serveur Linux

Mini-projet du module **Administration Sécurisée et Forensics (ASF)**
ENSA Béni Mellal, filière Intelligence Artificielle et Cybersécurité, 2025-2026
Encadrant : Pr. TOUIL | Étudiante : LATOUTI Hassnae

> **Avertissement** : ce projet a été réalisé dans un laboratoire isolé (réseau Host-only, machines virtuelles personnelles) à des fins pédagogiques. Les techniques présentées ne doivent jamais être utilisées sur des systèmes sans autorisation explicite.

---

## Présentation

Durcissement d'un serveur Ubuntu, simulation d'une attaque SSH par force brute depuis Kali Linux (Hydra), puis analyse forensique des logs (IOC, chronologie) et réponse à incident.

## Environnement

| Machine | Système | IP | Rôle |
|---|---|---|---|
| VM1 | Ubuntu (serveur) | 192.168.56.10 | Victime |
| VM2 | Kali Linux 2024 | 192.168.56.20 | Attaquant |

Réseau : VMnet1 (Host-only)
Outils : `auditd`, `rsyslog`, `ufw`, `chrony`, `fail2ban`, `Hydra`, `netcat`

## Structure du dépôt

```
.
├── README.md
├── Mini-Projet_ASF.pdf          # Énoncé
└── rapport_ASF_v5_Hassnae.pdf   # Rapport forensique
```

## Étapes réalisées

### 1. Modèle de menace
Actifs critiques, surface d'attaque (SSH, comptes, services), scénario : **bruteforce SSH + élévation de privilèges**.

### 2. Hardening
- `PermitRootLogin no`, `MaxAuthTries 5`
- Pare-feu UFW (SSH limité à `192.168.56.0/24`)
- Mises à jour automatiques (`unattended-upgrades`)
- Sudo contrôlé, suppression des comptes inutiles

### 3. Journalisation
- `auth.log` / `rsyslog`
- `auditd` avec règles persistantes :

```bash
-w /etc/passwd  -p wa -k passwd_modif
-w /etc/shadow  -p wa -k shadow_modif
-w /etc/sudoers -p wa -k sudoers_modif
-a always,exit -F arch=b64 -S connect -k connexion_sortante
-a always,exit -F arch=b64 -S execve  -k exec_cmd
```
- Synchronisation NTP avec `chrony`

### 4. Simulation d'attaque (lab)
Bruteforce SSH (Hydra) → connexion avec le compte compromis → création d'un compte suspect → élévation de privilèges → connexion sortante inhabituelle → persistance par clé SSH.

### 5. Détection et analyse

```bash
grep 'Failed password' /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
grep 'Accepted password' /var/log/auth.log
grep -E 'useradd|new user' /var/log/auth.log
sudo ausearch -k passwd_modif
sudo ausearch -k connexion_sortante
sudo aureport -au
```

### 6. Réponse à incident
- Blocage de l'IP attaquante : `ufw deny from 192.168.56.20`
- Suppression du compte malveillant : `userdel -r backdoor`
- Changement des mots de passe, suppression de `authorized_keys` injectée
- Vérification des crons (`crontab -l`)
- Déploiement de Fail2Ban (`maxretry=3`, `bantime=3600`)

##  Indicateurs de compromission (IOC)

| Type | Valeur |
|---|---|
| IP source | 192.168.56.20 |
| Compte compromis | alice (uid 1001, sudo) |
| Compte malveillant | backdoor (uid 1002, sudo) |
| Persistance | `/root/.ssh/authorized_keys` |
| Connexion sortante | 192.168.56.20:8080 |
| Fichier sensible lu | `/etc/shadow` |

##  Recommandations

1. Mots de passe forts (`libpam-pwquality`, 12 caractères minimum)
2. `PasswordAuthentication no` (clés SSH uniquement)
3. Fail2Ban sur tous les serveurs SSH
4. Centralisation des logs (SIEM)
5. Audits réguliers des comptes, sudo et clés SSH
6. Moindre privilège via `visudo`
7. Segmentation réseau et bastion SSH
8. Surveillance du trafic sortant (reverse shell, exfiltration)

## Rapport complet

 [`docs/rapport_ASF_v5_Hassnae.pdf`](docs/rapport_ASF_v5_Hassnae.pdf)

## Auteure

**LATOUTI Hassnae**, élève ingénieure IA & Cybersécurité, ENSA Béni Mellal
