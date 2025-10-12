# 📘 Nebula VPN – Guide de configuration client et Lighthouse

Ce guide explique comment configurer **Nebula** sur un VPS (Lighthouse) et générer les certificats pour les clients.

---

## 📋 Prérequis

- VPS avec Nebula installé (version ≥ 1.9.7 recommandée)
- `nebula-cert` disponible sur le VPS
- Certificat et clé CA (`ca.crt` et `ca.key`) déjà créés avec :
  ```bash
  nebula-cert ca -name "HombeLab"
  ```

---

## 📁 Arborescence recommandée

```
appdata/
└── nebula/
    └── config/
        ├── ca.crt
        ├── ca.key
        ├── lighthouse1.crt
        ├── lighthouse1.key
        ├── client1.crt
        ├── client1.key
        └── config.yml
```

---

## 🚀 Configuration du Lighthouse (VPS)

### 1. Générer le certificat du Lighthouse

Si ce n'est pas déjà fait, créez le certificat pour votre serveur Lighthouse :

```bash
nebula-cert sign \
  -name lighthouse1 \
  -ip 10.10.10.1/24 \
  -ca-crt ./ca.crt \
  -ca-key ./ca.key \
  -out-crt ./lighthouse1.crt \
  -out-key ./lighthouse1.key
```

### 2. Configuration du Lighthouse (`config.yml`)

Exemple de configuration pour le serveur Lighthouse :

```yaml
pki:
  ca: /path/to/appdata/nebula/config/ca.crt
  cert: /path/to/appdata/nebula/config/lighthouse1.crt
  key: /path/to/appdata/nebula/config/lighthouse1.key

static_host_map: {}

lighthouse:
  am_lighthouse: true
  serve_dns: false
  interval: 60

listen:
  host: "::"
  port: 4242

punchy:
  punch: true

tun:
  dev: nebula1
  drop_local_broadcast: false
  drop_multicast: false
  tx_queue: 500
  mtu: 1300

logging:
  level: info
  format: text

firewall:
  conntrack:
    tcp_timeout: 12m
    udp_timeout: 3m
    default_timeout: 10m

  outbound:
    - port: any
      proto: any
      host: any

  inbound:
    - port: any
      proto: any
      host: any
```

### 3. Démarrer le Lighthouse

```bash
sudo nebula -config /path/to/appdata/nebula/config/config.yml
```

---

## 👥 Configuration des clients

### 1. Générer un certificat client

Pour chaque nouveau client, générez un certificat avec une IP VPN unique :

```bash
nebula-cert sign \
  -name client1 \
  -ip 10.10.10.2/24 \
  -out-crt ./client1.crt \
  -out-key ./client1.key \
  -ca-crt ./ca.crt \
  -ca-key ./ca.key
```

⚠️ **Important** : Chaque client doit avoir un nom et une IP VPN uniques dans le réseau.

**Exemples pour d'autres clients :**
- Client2 : `10.10.10.3/24`
- Client3 : `10.10.10.4/24`
- etc.

### 2. Transférer les fichiers au client

Copiez les fichiers suivants sur la machine cliente :
- `ca.crt`
- `client1.crt`
- `client1.key`

### 3. Configuration du client (`config.yml`)

Créez un fichier de configuration sur le client :

```yaml
pki:
  ca: /path/to/ca.crt
  cert: /path/to/client1.crt
  key: /path/to/client1.key

static_host_map:
  "10.10.10.1": ["82.165.195.214:4242"]

lighthouse:
  am_lighthouse: false
  interval: 60
  hosts:
    - "10.10.10.1"

listen:
  host: "::"
  port: 4242

punchy:
  punch: true

tun:
  dev: nebula1 //utun sur macOs par exemple utun9, utun10 ...
  drop_local_broadcast: false
  drop_multicast: false
  tx_queue: 500
  mtu: 1300

logging:
  level: info
  format: text

firewall:
  conntrack:
    tcp_timeout: 12m
    udp_timeout: 3m
    default_timeout: 10m

  outbound:
    - port: any
      proto: any
      host: any

  inbound:
    - port: any
      proto: any
      host: any
```

### 4. Démarrer Nebula sur le client

```bash
sudo nebula -config /path/to/config.yml
```

---

## ✅ Vérification

### Tester la connectivité

Depuis un client, testez la connexion au Lighthouse :

```bash
ping 10.10.10.1
```

Depuis un autre client, testez la connexion peer-to-peer :

```bash
ping 10.10.10.2
```

### Vérifier l'état de Nebula

```bash
# Voir les logs
sudo journalctl -u nebula -f

# Vérifier l'interface réseau
ip addr show nebula1
```

---

## 🔧 Dépannage

### Le client ne peut pas se connecter

1. Vérifiez que le port 4242/UDP est ouvert sur le Lighthouse
2. Vérifiez que les certificats sont valides et correspondent
3. Vérifiez les chemins des fichiers dans `config.yml`

### Erreur "certificate is not valid"

- Assurez-vous que tous les certificats ont été signés avec le même CA
- Vérifiez les dates d'expiration des certificats

### Pas de connectivité entre clients

- Vérifiez que `punchy.punch: true` est activé sur tous les clients
- Vérifiez que les règles de firewall autorisent le trafic

---

## 📝 Notes importantes

- **Sécurité** : Gardez `ca.key` en sécurité et ne le distribuez jamais aux clients
- **Backup** : Sauvegardez régulièrement vos certificats CA
- **Rotation** : Les certificats Nebula ont une durée de validité limitée (par défaut 1 an)
- **Performances** : Ajustez le MTU selon votre réseau (1300 est une valeur sûre)

---

## 🔗 Ressources

- [Documentation officielle Nebula](https://nebula.defined.net/docs/)
- [GitHub Nebula](https://github.com/slackhq/nebula)
- [Guide de démarrage rapide](https://nebula.defined.net/docs/guides/quick-start/)

---

**Version du guide** : 1.0  
**Dernière mise à jour** : Octobre 2025
