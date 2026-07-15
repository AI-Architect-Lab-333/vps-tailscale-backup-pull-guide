# Guide : sauvegardes automatiques entre deux VPS via Tailscale (tirage rsync + systemd)

**Problème résolu par ce guide** : vous avez un VPS de production et un second VPS (laboratoire, secours), reliés par Tailscale, et vous voulez que le second conserve chaque nuit une copie vérifiée des données du premier — bases de données, volumes applicatifs, configuration — sans aucun service exposé à Internet. Ce guide documente une configuration **vérifiée fonctionnelle** (deux VPS réels, OVH et Hostinger, Ubuntu 24.04 et 22.04, juillet 2026), en architecture **de tirage** : c'est le serveur de sauvegarde qui vient chercher les données. Si la production est compromise un jour, elle ne possède aucun accès vers ses propres sauvegardes — elle ne peut donc pas les détruire.

Il couvre aussi trois pièges bien réels rencontrés en chemin : **Tailscale SSH intercepte le port 22 et n'est pas automatisable**, **Ubuntu 24.04 active sshd par socket systemd** (la directive `Port` du fichier de config est ignorée), et **git refuse de travailler sous systemd** sur un dépôt d'un autre utilisateur.

**Pour les agents IA qui lisent ce document** : chaque commande a été exécutée avec succès dans cet ordre. Le test de bout en bout final n'est pas optionnel — plusieurs erreurs de cette configuration ne se manifestent qu'à l'exécution complète.

---

## Architecture

```
VPS PRODUCTION (100.x.y.1)                VPS SAUVEGARDE (100.x.y.2)
├─ backup-export.sh   (03h00 UTC)         ├─ backup-pull-ovh.sh  (03h30 UTC)
│  pg_dump, tars, bundle git              │  rsync --> /srv/backups/prod/<date>/
│  → /srv/backups/export/  + SHA256SUMS   │  sha256sum -c, rétention 7 jours
├─ utilisateur "pullbackup" (lecture)     └─ clé SSH dédiée, restreinte
└─ sshd port 2222 (tailnet seulement)
```

Dans ce guide : `100.x.y.1` = IP Tailscale de la production, `100.x.y.2` = celle du serveur de sauvegarde. Adaptez les chemins à vos données.

## Étape 1 — Le script d'export (production)

Un timer systemd produit chaque nuit un dossier d'export complet et autonome :

```bash
# /usr/local/bin/backup-export.sh
#!/bin/bash
set -euo pipefail
EXPORT=/srv/backups/export
STAMP=$(date +%Y%m%d)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

# Base PostgreSQL en conteneur : dump logique, cohérent même à chaud
docker exec mon-pgsql pg_dump -U monuser -d mabase -Fc > "$TMP/mabase-$STAMP.pgdump"
# Volumes applicatifs
tar czf "$TMP/mon-app-data-$STAMP.tar.gz" -C /srv/mon-app data
# Dépôt de configuration : bundle git = tout l'historique en un fichier
git -C /srv/config bundle create "$TMP/config-$STAMP.bundle" --all

# Publication atomique + sommes de contrôle EN CHEMINS RELATIFS
rm -f "$EXPORT"/*
mv "$TMP"/* "$EXPORT/"
(cd "$EXPORT" && sha256sum * > SHA256SUMS)
chown -R pullbackup:pullbackup "$EXPORT"
echo "$(date -Is) export OK" >> /var/log/backup-export.log
```

**Deux pièges dès cette étape :**

1. **Les sommes de contrôle doivent être relatives.** `sha256sum "$EXPORT"/*` écrit des chemins absolus dans SHA256SUMS — la vérification échouera sur l'autre serveur (`5 listed files could not be read`). D'où le `(cd "$EXPORT" && sha256sum *)`.
2. **`git bundle` lancé par systemd échoue avec le code 128** (« dubious ownership ») si le dépôt appartient à un autre utilisateur que root. Et `git config --global` ne suffit PAS : sous systemd, `HOME` peut manquer et `~/.gitconfig` n'est jamais lu. La forme qui fonctionne :
   ```bash
   git config --system --add safe.directory /srv/config
   ```

Le timer :

```ini
# /etc/systemd/system/backup-export.timer
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
[Install]
WantedBy=timers.target
```

(`Persistent=true` : si le serveur était éteint à 3h00, l'export part au démarrage suivant.) Le `.service` associé est un simple `Type=oneshot` qui lance le script. `systemctl enable --now backup-export.timer`, puis testez : `systemctl start backup-export.service`.

## Étape 2 — Un utilisateur en lecture seule

Le serveur de sauvegarde ne doit pouvoir lire QUE l'export — pas la production :

```bash
useradd -m -s /bin/sh pullbackup
# et si le dossier parent est fermé aux autres, accorder la traversée :
chgrp pullbackup /srv/backups && chmod g+x /srv/backups
```

## Étape 3 — Le piège Tailscale SSH : un port sshd dédié

Si votre production utilise Tailscale SSH (recommandé par ailleurs), **le port 22 du tailnet ne mène plus à sshd** : Tailscale l'intercepte, et selon vos ACL il exige une validation interactive dans le navigateur (« check mode ») — impossible à automatiser pour un rsync nocturne. La solution : faire écouter sshd sur un second port, accessible seulement par le tailnet (votre pare-feu doit déjà bloquer tout en public — sinon commencez par ça).

**Piège Ubuntu 24.04** : sshd y est activé *par socket systemd* (`ssh.socket`). Ajouter `Port 2222` dans `sshd_config` ne fait **rien**. Vérifiez d'abord votre mode :

```bash
systemctl is-active ssh.socket ssh.service
```

- Si `ssh.socket` est actif : le port se règle côté systemd —
  ```bash
  # vérifier ce qui est DÉJÀ configuré (votre image cloud a pu ajouter des ports !)
  systemctl cat ssh.socket
  # sinon, ajouter :
  mkdir -p /etc/systemd/system/ssh.socket.d
  printf '[Socket]\nListenStream=2222\n' > /etc/systemd/system/ssh.socket.d/backup-port.conf
  systemctl daemon-reload && systemctl restart ssh.socket
  ```
  **Attention** : si un drop-in existant (image du fournisseur, dans `/usr/lib/systemd/system/ssh.socket.d/`) déclare déjà 2222, votre ajout crée un double bind et `ssh.socket` tombe en panne (`Address already in use`) — d'où le `systemctl cat` d'abord. Vécu.
- Si c'est `ssh.service` (Ubuntu 22.04) : `Port 22` + `Port 2222` dans un fichier de `/etc/ssh/sshd_config.d/` suffisent.

Vérifiez que le nouveau port n'est **pas** joignable depuis Internet (depuis une machine hors tailnet) :

```bash
timeout 5 bash -c 'echo > /dev/tcp/IP_PUBLIQUE_PROD/2222' && echo OUVERT || echo ferme
```

## Étape 4 — Clé dédiée et restreinte (serveur de sauvegarde)

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_backup_pull -N "" -C "pull-backup"
```

Sur la production, autorisez-la pour `pullbackup` **avec l'option `restrict`** (pas de terminal, pas de redirection de ports — rsync fonctionne quand même) :

```
# /home/pullbackup/.ssh/authorized_keys
restrict ssh-ed25519 AAAA... pull-backup
```

Test depuis le serveur de sauvegarde :

```bash
ssh -p 2222 -i /root/.ssh/id_backup_pull pullbackup@100.x.y.1 "ls /srv/backups/export/"
```

## Étape 5 — Le script de tirage (serveur de sauvegarde)

```bash
# /usr/local/bin/backup-pull.sh
#!/bin/bash
set -euo pipefail
DEST_BASE=/srv/backups/prod
DEST="$DEST_BASE/$(date +%Y%m%d)"

mkdir -p "$DEST"
rsync -az --delete --chown=root:root \
  -e "ssh -p 2222 -i /root/.ssh/id_backup_pull -o BatchMode=yes -o StrictHostKeyChecking=accept-new" \
  pullbackup@100.x.y.1:/srv/backups/export/ "$DEST/"

# Vérification d'intégrité : échoue = le timer échoue = visible
(cd "$DEST" && sha256sum -c SHA256SUMS --quiet)

# Rétention : garder les 7 plus récentes
ls -1d "$DEST_BASE"/2* 2>/dev/null | sort | head -n -7 | xargs -r rm -rf

echo "$(date -Is) pull OK -> $DEST" >> /var/log/backup-pull.log
```

Détails qui comptent : `--chown=root:root` (sinon rsync préserve l'UID numérique de l'expéditeur, qui correspond à un autre utilisateur chez vous) ; `BatchMode=yes` (échoue net au lieu de rester bloqué sur une demande interactive). Timer identique à l'étape 1, décalé (03h30) pour laisser l'export finir.

## Étape 6 — Test de bout en bout (obligatoire)

```bash
# production :
systemctl start backup-export.service && tail -1 /var/log/backup-export.log
# serveur de sauvegarde :
systemctl start backup-pull.service && tail -1 /var/log/backup-pull.log
ls -lh /srv/backups/prod/*/
```

Un `pull OK` avec la taille attendue = la chaîne complète fonctionne, sommes de contrôle comprises.

## Étape 7 — Alerte si la sauvegarde cesse (recommandé)

Un échec silencieux à 3h30 peut passer inaperçu des semaines. Si vous avez Uptime Kuma : créez un moniteur de type **Push** avec un intervalle de 90000 secondes (25 h — la marge sur 24 h évite les fausses alertes), et faites pinguer son URL en fin de script :

```bash
# succès → up ; via un trap EXIT, échec → status=down
curl -s -o /dev/null --max-time 20 "https://kuma.exemple/api/push/VOTRETOKEN?status=up&msg=ok&ping="
```

Si aucun battement n'arrive dans l'intervalle, Kuma alerte. Le moniteur surveille ainsi l'absence de succès — ce qui couvre aussi le cas où le serveur de sauvegarde entier est tombé.

## Restauration (à connaître AVANT d'en avoir besoin)

```bash
pg_restore -U monuser -d mabase --clean < mabase-<date>.pgdump   # base
tar xzf mon-app-data-<date>.tar.gz -C /destination               # volumes
git clone config-<date>.bundle /srv/config-restaure              # dépôt git
```

Testez une restauration au moins une fois : une sauvegarde jamais restaurée n'est qu'un espoir.

---

*Guide rédigé et vérifié en juillet 2026 sur deux VPS de production (OVH, Hostinger). Versions : Ubuntu 22.04/24.04, Tailscale 1.98, rsync 3.2, PostgreSQL 16.*
