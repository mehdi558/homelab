# 📦 Configuration d'un montage NFS automatique et résilient sous Linux

Ce guide explique comment monter un partage NFS (ex : depuis un NAS ou un Mac) de manière persistante, performante et auto-reconnectée, même après une perte temporaire de connexion réseau.

**Systèmes compatibles** : Ubuntu / OMV / Proxmox VM / Debian

---

## 🧭 Objectif

Monter automatiquement un partage NFS accessible à `192.168.1.145:/Volumes/media_pool` dans le dossier local `/mnt/media_share`, pour l'utiliser avec qBittorrent, Plex, Radarr, Sonarr, ou tout autre conteneur Docker.

---

## ⚙️ Étape 1 — Installer le client NFS

```bash
sudo apt update
sudo apt install nfs-common -y
```

---

## 📂 Étape 2 — Créer le dossier de montage

```bash
sudo mkdir -p /mnt/media_share
```

---

## 🧩 Étape 3 — Modifier le fichier `/etc/fstab`

Ouvre le fichier `fstab` :

```bash
sudo nano /etc/fstab
```

Ajoute cette ligne à la fin :

### Configuration standard (équilibrée)

```bash
# Montage NFS résilient pour Docker / Plex / qBittorrent
192.168.1.145:/Volumes/media_pool  /mnt/media_share  nfs  defaults,_netdev,noauto,x-systemd.automount,nfsvers=3,nolock,uid=1000,gid=1000,rsize=262144,wsize=262144,async,timeo=14,retrans=3,intr  0  0
```

### Configuration optimisée pour qBittorrent (recommandée)

```bash
# Montage NFS optimisé pour qBittorrent - Priorité : performance et résilience
192.168.1.145:/Volumes/media_pool  /mnt/media_share  nfs  nfsvers=3,nolock,rsize=1048576,wsize=1048576,async,noactime,nodiratime,timeo=600,retrans=2,soft,_netdev,noauto,x-systemd.automount  0  0
```

### 🎯 Différences clés pour qBittorrent

| Option | Standard | Optimisé qBittorrent | Raison |
|--------|----------|---------------------|---------|
| `rsize/wsize` | 262144 (256K) | **1048576 (1M)** | Blocs plus gros = moins d'appels NFS pour gros fichiers |
| `timeo` | 14 (1.4s) | **600 (60s)** | Plus tolérant aux ralentissements réseau |
| `retrans` | 3 | **2** | Évite surcharge réseau en cas de timeout |
| `soft` | ❌ absent | **✅ ajouté** | qBittorrent ne se bloque pas si NFS est HS |
| `noactime` | ❌ absent | **✅ ajouté** | Pas de MAJ du temps d'accès = moins d'écritures |
| `nodiratime` | ❌ absent | **✅ ajouté** | Idem pour les dossiers |
| `intr` | ✅ présent | ❌ retiré | Obsolète avec `soft` |

### 💡 Pourquoi ces changements ?

**Pour qBittorrent spécifiquement :**

1. **`rsize/wsize=1048576`** : qBittorrent écrit des fichiers volumineux séquentiellement → blocs de 1 MB réduisent drastiquement les appels réseau

2. **`soft` au lieu de `hard`** : Si NFS plante, qBittorrent reçoit une erreur au lieu de geler complètement. Crucial car qBittorrent peut mettre en pause les téléchargements et réessayer.

3. **`timeo=600`** : 60 secondes de délai avant timeout. qBittorrent a un cache disque interne qui peut absorber ces délais.

4. **`noactime,nodiratime`** : qBittorrent accède fréquemment aux fichiers → éviter de mettre à jour le "dernier accès" économise énormément d'écritures inutiles.

5. **`retrans=2`** : Avec `soft` et un long timeout, 2 tentatives suffisent (au lieu de 3) pour éviter de saturer le réseau.

### 💡 Détails des options

| Option | Description |
|--------|-------------|
| `nfsvers=3` | Forcer NFS v3 (requis pour les serveurs macOS) |
| `nolock` | Évite les erreurs de verrouillage sur systèmes multi-clients |
| `rsize/wsize=262144` | Taille des blocs lecture/écriture (256 K) → bon compromis perf/stabilité |
| `async` | Améliore la vitesse d'écriture |
| `_netdev` | Attend que le réseau soit prêt avant de monter |
| `x-systemd.automount` | Monte automatiquement à la première utilisation |
| `noauto` | Évite de bloquer le démarrage si le serveur est éteint |
| `intr,timeo=14,retrans=3` | Rend le montage tolérant aux coupures réseau |

---

## 🔄 Étape 4 — Recharger systemd et tester le montage

Recharger les services système :

```bash
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
```

Monter immédiatement :

```bash
sudo mount -a
```

Vérifier le résultat :

```bash
df -h | grep media_share
```

---

## ✅ Étape 5 — Vérification et test des droits

Affiche les fichiers :

```bash
ls -l /mnt/media_share
```

Teste les permissions :

```bash
touch /mnt/media_share/test.txt
rm /mnt/media_share/test.txt
```

✅ Si ces commandes fonctionnent, les permissions sont bonnes pour ton utilisateur et Docker.

---

## 🔁 Étape 6 — Test de reconnexion automatique

1. Coupe le réseau ou éteins temporairement ton serveur NFS (`192.168.1.145`)
2. Réactive-le
3. Accède à nouveau au dossier :

```bash
ls /mnt/media_share
```

💡 Grâce à `x-systemd.automount`, le partage se reconnecte automatiquement dès qu'il est de nouveau disponible.

---

## 🐳 Étape 7 — Utilisation dans Docker

Exemple pour qBittorrent :

```yaml
volumes:
  - /mnt/media_share:/downloads
```

Ainsi, qBittorrent télécharge directement sur ton partage NFS sans copie locale.

---

## 🧹 Commandes utiles

### Démonter manuellement

```bash
sudo umount /mnt/media_share
```

### Remonter tous les partages

```bash
sudo mount -a
```

### Vérifier les erreurs NFS

```bash
sudo dmesg | tail -n 20
```

---

## 📘 Résumé de la configuration

| Élément | Valeur |
|---------|--------|
| Partage NFS | `192.168.1.145:/Volumes/media_pool` |
| Point de montage | `/mnt/media_share` |
| UID/GID | `1000:1000` |
| Protocole | NFS v3 |
| Auto-reconnexion | ✅ via `x-systemd.automount` |
| Taille des blocs | `rsize=262144`, `wsize=262144` |

---

## 💡 Avantages de cette configuration

- ⚙️ **Pas besoin d'identifiants** (contrairement au SMB)
- ⚡ **Performances supérieures** sur fichiers volumineux
- 🔁 **Reconnexion automatique** après coupure réseau
- 🧱 **Compatible Docker** / Plex / qBittorrent
- 🚀 **Sans blocage au démarrage** du système

---

## 🧠 Astuce : Surveillance automatique

Surveille la disponibilité du montage via `Netdata`, `Beszel` ou un script simple.

### Script de surveillance basique

Crée le fichier `/usr/local/bin/check-nfs.sh` :

```bash
#!/bin/bash
if ! mountpoint -q /mnt/media_share; then
  echo "⚠️ Montage NFS perdu, tentative de reconnexion..."
  sudo mount -a
fi
```

Rends-le exécutable :

```bash
sudo chmod +x /usr/local/bin/check-nfs.sh
```

### Planification avec cron

Édite la crontab :

```bash
sudo crontab -e
```

Ajoute cette ligne pour vérifier toutes les 10 minutes :

```bash
*/10 * * * * /usr/local/bin/check-nfs.sh
```

💡 **Pour une solution plus avancée avec logs et gestion des erreurs**, consulte le guide complet de surveillance NFS.

---

## 🆘 Dépannage

### Le montage ne fonctionne pas

Vérifier que le serveur NFS est accessible :

```bash
showmount -e 192.168.1.145
```

### Erreurs de permissions

Vérifier l'UID/GID de ton utilisateur :

```bash
id
```

Si différent de 1000, ajuste la ligne dans `/etc/fstab` avec les bonnes valeurs.

### Montage bloqué (stale NFS)

Forcer le démontage :

```bash
sudo umount -f /mnt/media_share
sudo mount -a
```

---

## 📚 Ressources complémentaires

- [Documentation NFS Ubuntu](https://ubuntu.com/server/docs/service-nfs)
- [Guide NFS Options](https://linux.die.net/man/5/nfs)
- [Systemd Mount Units](https://www.freedesktop.org/software/systemd/man/systemd.mount.html)

---

*Dernière mise à jour : 2 novembre 2025*
