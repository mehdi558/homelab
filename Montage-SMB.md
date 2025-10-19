# 📦 Configuration d’un montage SMB automatique et résilient sous Linux (Ubuntu / OMV / Proxmox VM)

Ce guide explique comment monter un dossier SMB (Windows ou NAS) de manière **persistante et auto-reconnectée**, même après une perte temporaire de connexion réseau.

---

## 🧭 Objectif

Monter automatiquement un partage SMB accessible à `//192.168.1.145/media_pool`  
sur ton serveur local dans `/mnt/media_share`, pour l’utiliser dans des applications Docker (qBittorrent, Radarr, Sonarr, etc.).

---

## ⚙️ Étape 1 — Créer le dossier de montage

```bash
sudo mkdir -p /mnt/media_share
```

---

## 🧰 Étape 2 — Créer le fichier d’identifiants sécurisés

Créer un fichier pour stocker les identifiants SMB de manière sécurisée :

```bash
sudo mkdir -p /etc/samba
sudo nano /etc/samba/creds-media
```

Contenu du fichier :
```
username="username"
password="password"
```

Protéger le fichier :
```bash
sudo chmod 600 /etc/samba/creds-media
```

---

## 🧩 Étape 3 — Modifier le fichier `/etc/fstab`

Ouvre le fichier `fstab` :
```bash
sudo nano /etc/fstab
```

Ajoute cette ligne à la fin :

```
# Montage SMB résilient pour Docker / Plex / Radarr / Sonarr
//192.168.1.145/media_pool /mnt/media_share cifs credentials=/etc/samba/creds-media,iocharset=utf8,file_mode=0777,dir_mode=0777,nounix,noserverino,vers=3.1.1,_netdev,x-systemd.automount,retry=5,soft,timeo=30,cache=loose 0 0
```

### 💡 Détails des options :
| Option | Description |
|--------|--------------|
| `credentials=` | Chemin vers le fichier contenant les identifiants SMB |
| `_netdev` | Attend que le réseau soit disponible avant de monter |
| `x-systemd.automount` | Monte le partage à la demande et le reconnecte automatiquement |
| `vers=3.1.1` | Force le protocole SMB 3 moderne |
| `soft,timeo=30` | Évite les blocages en cas de déconnexion |
| `file_mode=0777,dir_mode=0777` | Permet à tous les conteneurs Docker d’écrire dessus |

---

## 🔄 Étape 4 — Recharger systemd et tester le montage

Recharger les services système :
```bash
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
```

Monter le partage immédiatement :
```bash
sudo mount -a
```

Vérifier le résultat :
```bash
df -h | grep media_share
```

---

## ✅ Étape 5 — Vérification et test des droits

Afficher les fichiers :
```bash
ls -l /mnt/media_share
```

Tester les permissions :
```bash
touch /mnt/media_share/test.txt
rm /mnt/media_share/test.txt
```

✅ Si cela fonctionne, ton utilisateur et Docker peuvent lire/écrire dans le partage.

---

## 🔁 Étape 6 — Test de reconnexion automatique

1. **Déconnecte** temporairement ton NAS ou désactive le réseau.
2. **Réactive** le réseau.
3. Accède à nouveau au dossier :
   ```bash
   ls /mnt/media_share
   ```
   💡 Grâce à `x-systemd.automount`, le montage sera **automatiquement rétabli**.

---

## 🐳 Étape 7 — Utilisation dans Docker

Exemple pour qBittorrent :
```yaml
volumes:
  - /mnt/media_share:/downloads
```

Ainsi, tous les fichiers téléchargés seront écrits directement dans le partage SMB.

---

## 🧹 Commandes utiles

- Démonter manuellement :
  ```bash
  sudo umount /mnt/media_share
  ```
- Remonter tous les partages :
  ```bash
  sudo mount -a
  ```
- Vérifier les erreurs dans les logs :
  ```bash
  sudo dmesg | tail -n 20
  ```

---

## 📘 Résumé

| Élément | Valeur |
|----------|--------|
| Partage SMB | `//192.168.1.145/media_pool` |
| Point de montage | `/mnt/media_share` |
| Fichier credentials | `/etc/samba/creds-media` |
| Protocole | SMB 3.1.1 |
| Auto-reconnexion | ✅ (via `x-systemd.automount`) |

---

💡 **Avantages :**
- Résilient aux pertes de connexion  
- Monte automatiquement au démarrage ou à la demande  
- Compatible avec Docker, Plex, Radarr, Sonarr et OMV  

---

> 🧠 Astuce : tu peux surveiller le montage avec une alerte `Netdata` ou `Beszel` pour détecter une déconnexion SMB automatiquement.
