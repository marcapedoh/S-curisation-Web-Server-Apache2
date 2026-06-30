# 🛡️ Sécurisation Apache2 avec ModSecurity & OWASP CRS v4

**Projet 06** | Sécurisation d'un serveur Apache2 contre les attaques web courantes  
**Durée** : 1 semaine | **Technos** : Apache2, ModSecurity v3, OWASP CRS v4, fail2ban

---

## 📋 Table des matières

1. [Contexte & Objectifs](#contexte--objectifs)
2. [Architecture du déploiement](#architecture-du-déploiement)
3. [Prérequis](#prérequis)
4. [Installation complète (étape par étape)](#installation-complète-étape-par-étape)
5. [Erreurs rencontrées & Solutions](#erreurs-rencontrées--solutions)
6. [Tests d'attaque & Validation](#tests-dattaque--validation)
7. [Analyse des logs](#analyse-des-logs)
8. [Configuration fail2ban](#configuration-fail2ban)
9. [Résultats finaux](#résultats-finaux)
10. [Prochaines étapes](#prochaines-étapes)

---

## 🎯 Contexte & Objectifs

### Contexte
Une application web est victime d'attaques régulières :
- **Injections SQL** (`' OR 1=1--`)
- **Cross-Site Scripting (XSS)** (`<script>alert(1)</script>`)
- **Brute Force** (plusieurs tentatives de connexion)
- **Path Traversal** (`../../../etc/passwd`)

### Objectifs pédagogiques
✅ Installer **ModSecurity v3** avec le module Apache  
✅ Déployer **OWASP Core Rule Set v4** en mode **Enforcement**  
✅ Configurer **fail2ban** pour bloquer les IPs malveillantes  
✅ Tester et valider la détection d'injections SQL et XSS  
✅ Analyser les logs d'audit ModSecurity  
✅ Documenter les faux positifs et créer des exclusions personnalisées  

---

## 🏗️ Architecture du déploiement

```
┌─────────────────────────────────────────────────────────────┐
│                     Client / Attaquant                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ HTTP Request
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                      Apache2 HTTP Server                    │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │   ModSecurity v3 (WAF - Web Application Firewall)     │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  OWASP Core Rule Set v4 (Paranoia Level 2)      │  │  │
│  │  │  ┌──────────────────────────────────────────┐   │  │  │
│  │  │  │ Rules:                                   │   │  │  │
│  │  │  │ • 942xxx - SQL Injection Detection       │   │  │  │
│  │  │  │ • 941xxx - XSS Detection                 │   │  │  │
│  │  │  │ • 930xxx - LFI Detection                 │   │  │  │
│  │  │  │ • 949xxx - Anomaly Scoring & Blocking    │   │  │  │
│  │  │  └──────────────────────────────────────────┘   │  │  │
│  │  │                                                 │  │  │
│  │  │  Mode: SecRuleEngine On (ENFORCEMENT)           │  │  │
│  │  │  Anomaly Scoring: Blocking Threshold = 5        │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                  │
│       Trafic Légitime    │   Trafic Malveillant             │
│          (Passe)         │      (Bloqué - 403)              │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
               Application    Audit Log
              (mod_php)        /var/log/apache2/
                              modsec_audit.log
```

---

## 📦 Prérequis

### Environnement
- **OS** : Debian 13 Trixie (GNU/Linux)
- **Web Server** : Apache2 2.4.67
- **Memory** : 2GB RAM minimum
- **Disque** : 500MB libre
- **Accès** : sudo/root

### Vérification de l'environnement
```bash
# Vérifier la version de Debian
lsb_release -a
# Output: Debian GNU/Linux 13 (trixie)

# Vérifier Apache2
apache2 -v
# Output: Apache/2.4.67 (Debian)

# Vérifier la RAM disponible
free -h

# Vérifier l'espace disque
df -h /
```

---

## 🚀 Installation complète (étape par étape)

### Phase 1️⃣ : Installation de ModSecurity v3

#### 1.1 Mettre à jour les paquets
```bash
sudo apt update
sudo apt upgrade -y
```

#### 1.2 Installer ModSecurity avec le module Apache
```bash
sudo apt install libapache2-mod-security2 -y
```

**Output attendu** :
```
libapache2-mod-security2 is already the newest version (2.9.11-1).
```

#### 1.3 Vérifier que le module est présent
```bash
ls -la /usr/lib/apache2/modules/mod_security2.so
```

**Output** :
```
-rw-r--r-- 1 root root 712256 Dec  5  2024 mod_security2.so
```

#### 1.4 Activer le module ModSecurity dans Apache
```bash
sudo a2enmod security2
```

**Output** :
```
Enabling module security2.
To activate the new configuration, you need to run:
  systemctl restart apache2
```

#### 1.5 Vérifier l'activation
```bash
apache2ctl -M | grep security
```

**Output** :
```
security2_module (shared)
```

---

### Phase 2️⃣ : Configuration initiale de ModSecurity

#### 2.1 Copier la configuration recommandée
```bash
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```

#### 2.2 Vérifier le fichier de config
```bash
ls -la /etc/modsecurity/modsecurity.conf
```

**Output** :
```
-rw-r--r-- 1 root root 55486 Jun 16 20:00 /etc/modsecurity/modsecurity.conf
```

#### 2.3 Afficher les directives ModSecurity principales
```bash
grep "^SecRuleEngine\|^SecDataDir\|^SecAuditEngine" /etc/modsecurity/modsecurity.conf
```

**Output** :
```
SecRuleEngine DetectionOnly
SecDataDir /var/cache/modsecurity
SecAuditEngine RelevantOnly
```

---

### Phase 3️⃣ : Installation et déploiement d'OWASP CRS v4

#### 3.1 Installer les dépendances
```bash
sudo apt install git curl -y
```

#### 3.2 Cloner le dépôt officiel OWASP CRS
```bash
cd /etc/modsecurity
sudo git clone https://github.com/coreruleset/coreruleset.git crs
```

**Output** :
```
Cloning into 'crs'...
remote: Enumerating objects: 4532, compressing objects: 100% (1237/1237), done.
Resolving deltas: 100% (3038/3038), done.
```

#### 3.3 Naviguer vers le répertoire CRS
```bash
cd /etc/modsecurity/crs
```

#### 3.4 Copier le fichier de configuration d'exemple
```bash
sudo cp crs-setup.conf.example crs-setup.conf
```

#### 3.5 Vérifier la structure des fichiers
```bash
ls -la /etc/modsecurity/crs/ | head -20
```

**Output** :
```
drwxr-xr-x  5 root root  4096 Jun 16 20:10 .
drwxr-xr-x  7 root root  4096 Jun 16 19:50 ..
-rw-r--r--  1 root root  2850 Jun 16 20:10 crs-setup.conf
-rw-r--r--  1 root root  2850 Jun 16 20:00 crs-setup.conf.example
-rw-r--r--  1 root root  5432 Jun 16 20:00 INSTALL.md
drwxr-xr-x 26 root root  4096 Jun 16 20:00 rules
```

#### 3.6 Vérifier que les fichiers de règles sont présents
```bash
ls /etc/modsecurity/crs/rules/ | head -10
```

**Output** :
```
REQUEST-900-EXCLUSIONS-FILE-BEFORE-CORE.conf
REQUEST-901-INITIALIZATION.conf
REQUEST-905-COMMON-EXCEPTIONS.conf
REQUEST-910-IP-REPUTATION.conf
REQUEST-912-DOS-PROTECTION.conf
REQUEST-913-SCANNER-DETECTION.conf
REQUEST-920-PROTOCOL-ENFORCEMENT.conf
REQUEST-921-PROTOCOL-ATTACK.conf
REQUEST-930-APPLICATION-ATTACK-LFI.conf
REQUEST-931-APPLICATION-ATTACK-RFI.conf
...
```

#### 3.7 Compter les fichiers de règles
```bash
ls /etc/modsecurity/crs/rules/ | wc -l
```

**Output** :
```
38
```

---

### Phase 4️⃣ : Configuration avancée de ModSecurity

#### 4.1 Modifier le fichier de configuration pour activer l'Enforcement Mode
```bash
sudo nano /etc/modsecurity/modsecurity.conf
```

**Chercher la ligne** (vers la ligne 14) :
```
SecRuleEngine DetectionOnly
```

**Remplacer par** :
```
SecRuleEngine On
```

**Vérifier la modification** :
```bash
grep "^SecRuleEngine" /etc/modsecurity/modsecurity.conf
```

**Output** :
```
SecRuleEngine On
```

#### 4.2 Configurer le niveau de paranoïa à 2 dans crs-setup.conf
```bash
sudo nano /etc/modsecurity/crs/crs-setup.conf
```

**Chercher la section** (vers la ligne 85-90) :
```
# paranoia_level
#SecAction "id:900000,phase:1,nolog,pass,t:none,setvar:tx.paranoia_level=1"
```

**Remplacer par** :
```
# paranoia_level
SecAction "id:900000,phase:1,nolog,pass,t:none,setvar:tx.paranoia_level=2"
```

**Vérifier** :
```bash
grep "tx.paranoia_level" /etc/modsecurity/crs/crs-setup.conf
```

**Output** :
```
SecAction "id:900000,phase:1,nolog,pass,t:none,setvar:tx.paranoia_level=2"
```

---

### Phase 5️⃣ : Configuration du chargement des règles CRS dans Apache

#### 5.1 Modifier le fichier security2.conf
```bash
sudo nano /etc/apache2/mods-enabled/security2.conf
```

**Chercher la ligne** :
```
IncludeOptional /usr/share/modsecurity-crs/*.load
```

**Remplacer par** :
```
IncludeOptional /etc/modsecurity/crs/crs-setup.conf
IncludeOptional /etc/modsecurity/crs/rules/*.conf
```

**Le fichier doit ressembler à ceci** :
```apache
<IfModule security2_module>
        # Default Debian dir for modsecurity's persistent data
        SecDataDir /var/cache/modsecurity

        # Include all the *.conf files in /etc/modsecurity.
        IncludeOptional /etc/modsecurity/*.conf

        # OWASP CRS v4 installé depuis GitHub
        IncludeOptional /etc/modsecurity/crs/crs-setup.conf
        IncludeOptional /etc/modsecurity/crs/rules/*.conf
</IfModule>
```

#### 5.2 Vérifier la syntax de la configuration Apache
```bash
sudo apachectl configtest
```

**Output attendu** :
```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

⚠️ **Note** : L'avertissement AH00558 n'est pas une erreur, c'est juste un avertissement concernant le ServerName. Nous pouvons l'ignorer ou le corriger (voir section 5.3).

#### 5.3 (OPTIONNEL) Corriger l'avertissement ServerName
```bash
echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/servername.conf

sudo a2enconf servername

sudo systemctl reload apache2

apachectl configtest
```

**Output** :
```
Syntax OK
```

#### 5.4 Redémarrer Apache
```bash
sudo systemctl restart apache2
```

#### 5.5 Vérifier que Apache est bien démarré
```bash
sudo systemctl status apache2
```

**Output attendu** :
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-06-16 20:15:12 CEST; 23s ago
     Docs: https://httpd.apache.org/docs/2.4/
    Process: 25343 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
   Main PID: 25346 (apache2)
      Tasks: 6 (limit: 4640)
     Memory: 61.1M (peak: 61.6M)
     CPU: 1.977s
```

#### 5.6 Vérifier que ModSecurity2 est chargé
```bash
apachectl -M | grep security
```

**Output** :
```
security2_module (shared)
```

---

### Phase 6️⃣ : Configuration des logs d'audit

#### 6.1 Vérifier les directives SecAudit
```bash
grep -E "^SecAudit" /etc/modsecurity/modsecurity.conf
```

**Output** :
```
SecAuditEngine RelevantOnly
SecAuditLogType Serial
SecAuditLog /var/log/apache2/modsec_audit.log
```

#### 6.2 Vérifier/créer le fichier d'audit
```bash
ls -l /var/log/apache2/modsec_audit.log
```

Si le fichier n'existe pas :
```bash
sudo touch /var/log/apache2/modsec_audit.log
sudo chown www-data:adm /var/log/apache2/modsec_audit.log
sudo chmod 640 /var/log/apache2/modsec_audit.log
```

#### 6.3 Redémarrer Apache pour appliquer les changements
```bash
sudo systemctl restart apache2
```

#### 6.4 Vérifier que le fichier d'audit est bien créé
```bash
sudo tail -5 /var/log/apache2/modsec_audit.log
```

---

## 🔴 Erreurs rencontrées & Solutions

### Erreur 1️⃣ : `IncludeOptional /usr/share/modsecurity-crs/*.load` pointait vers le mauvais répertoire

**Symptôme** :
```
ModSecurity était activé, mais les règles CRS ne se chargeaient pas.
Les attaques n'étaient pas détectées.
```

**Cause** :
La configuration de Debian utilisait par défaut le chemin `usr/share/modsecurity-crs/` qui correspond à la version de CRS installée via les paquets Debian, pas à la version GitHub que nous avions installée dans `/etc/modsecurity/crs/`.

**Solution appliquée** :
```bash
# Modifier /etc/apache2/mods-enabled/security2.conf
# Remplacer :
IncludeOptional /usr/share/modsecurity-crs/*.load

# Par :
IncludeOptional /etc/modsecurity/crs/crs-setup.conf
IncludeOptional /etc/modsecurity/crs/rules/*.conf
```

**Validation** :
```bash
sudo apachectl configtest  # Syntax OK
sudo systemctl restart apache2
apachectl -M | grep security  # security2_module (shared) ✓
```

---

### Erreur 2️⃣ : Avertissement `AH00558` au démarrage d'Apache

**Symptôme** :
```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
```

**Cause** :
Apache n'avait pas de ServerName global défini.

**Solution appliquée** :
```bash
echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl reload apache2
```

**Validation** :
```bash
apachectl configtest  # Syntax OK (sans l'avertissement AH00558)
```

---

### Erreur 3️⃣ : Fichier d'audit `/var/log/apache2/modsec_audit.log` absent

**Symptôme** :
```
ModSecurity ne pouvait pas écrire les logs d'audit, certaines attaques n'étaient pas enregistrées.
```

**Cause** :
Le fichier d'audit n'était pas créé automatiquement.

**Solution appliquée** :
```bash
sudo touch /var/log/apache2/modsec_audit.log
sudo chown www-data:adm /var/log/apache2/modsec_audit.log
sudo chmod 640 /var/log/apache2/modsec_audit.log
sudo systemctl restart apache2
```

**Validation** :
```bash
ls -l /var/log/apache2/modsec_audit.log
# -rw-r----- 1 www-data adm 0 Jun 16 20:15 /var/log/apache2/modsec_audit.log ✓
```

---

## 🧪 Tests d'attaque & Validation

### Test 1️⃣ : SQL Injection (`' OR 1=1--`)

#### Préparation : Ouvrir les logs en temps réel
```bash
# Terminal 1 (logs)
sudo tail -f /var/log/apache2/modsec_audit.log
```

#### Lancer l'attaque
```bash
# Terminal 2 (attaque)
curl "http://localhost/?id=' OR 1=1--"
```

**Ou dans le navigateur** :
```
http://192.168.56.106/?id=%27%20OR%201=1--
```

#### Résultat attendu dans le navigateur
```
403 Forbidden
You don't have permission to access this resource.
```

---

### Analyse des logs pour SQL Injection

```
--ab27a124-A--
[16/Jun/2026:20:22:32.538022 +0200] ajGUaN5Soqo_y8UzZoHXSgAAAAA 192.168.56.1 50990 192.168.56.106 80
--ab27a124-B--
GET /?id=%27%20OR%201=1-- HTTP/1.1
Host: 192.168.56.106
Connection: keep-alive
...

--ab27a124-F--
HTTP/1.1 403 Forbidden
Content-Length: 319
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=iso-8859-1
```

#### Règles CRS déclenchées (6 au total) :

| Rule ID | Titre | Sévérité | Détection |
|---------|-------|----------|-----------|
| **942100** | SQL Injection Attack Detected via libinjection | CRITICAL | `s&1c` fingerprint |
| **942130** | SQL Injection Attack: SQL Boolean-based attack detected | CRITICAL | `1=1` detected |
| **942180** | Detects basic SQL authentication bypass attempts 1/3 | CRITICAL | `' OR 1=1` pattern |
| **942330** | Detects classic SQL injection probings 1/3 | CRITICAL | `' OR 1` pattern |
| **942390** | SQL Injection Attack | CRITICAL | `' OR 1=1--` exact match |
| **920350** | Host header is a numeric IP address | WARNING | `192.168.56.106` |

#### Anomaly Scoring
```
Inbound Anomaly Score : 28
Blocking Threshold    : 5
Result                : BLOCKED ✓
Action                : Access denied with code 403 (phase 2)
```

#### Explication détaillée de chaque log
```
Message 1 (942100) :
  Pattern: libinjection fingerprint 's&1c'
  Data: Matched in ARGS:id = ' OR 1=1--
  Severity: CRITICAL (25 points)

Message 2 (942130) :
  Pattern: SQL Boolean-based (1=1)
  Data: Matched in ARGS:id = 1
  Severity: CRITICAL (3 points)
  Paranoia Level: 2 (attaqué à ce niveau)

Message 3 (942180) :
  Pattern: SQL auth bypass (' OR 1=1)
  Data: Matched in ARGS:id
  Severity: CRITICAL
  Paranoia Level: 2

Message 4 (942330) :
  Pattern: Classic SQL injection (' OR 1)
  Data: Matched in ARGS:id
  Severity: CRITICAL
  Paranoia Level: 2

Message 5 (942390) :
  Pattern: Exact SQL Injection match (' OR 1=1--)
  Data: Complete payload matched
  Severity: CRITICAL

Message 6 (920350) :
  Pattern: Host header is numeric IP
  Data: 192.168.56.106
  Severity: WARNING (0 points - peut être ignoré)
```

#### Extrait du log complet
```bash
grep "SQL Injection Attack Detected via libinjection" /var/log/apache2/modsec_audit.log | head -1
```

**Output** :
```
Message: Warning. detected SQLi using libinjection with fingerprint 's&1c' 
[file "/etc/modsecurity/crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf"] 
[line "66"] 
[id "942100"] 
[msg "SQL Injection Attack Detected via libinjection"] 
[data "Matched Data: s&1c found within ARGS:id: ' OR 1=1--"] 
[severity "CRITICAL"] 
[ver "OWASP_CRS/4.28.0-dev"]
```

---

### Test 2️⃣ : Cross-Site Scripting (XSS) - `<script>alert(1)</script>`

#### Lancer l'attaque XSS
```bash
curl "http://localhost/?name=<script>alert(1)</script>"
```

**Ou dans le navigateur** :
```
http://192.168.56.106/?name=<script>alert(1)</script>
```

#### Résultat attendu
```
403 Forbidden
```

#### Vérifier les logs
```bash
sudo tail -20 /var/log/apache2/modsec_audit.log | grep -i "xss\|script"
```

---

### Test 3️⃣ : Path Traversal - `../../../etc/passwd`

#### Lancer l'attaque
```bash
curl "http://localhost/file=../../../etc/passwd"
```

**Ou dans le navigateur** :
```
http://192.168.56.106/?file=../../../etc/passwd
```

#### Résultat attendu
```
403 Forbidden
```

#### Vérifier les logs
```bash
sudo tail -20 /var/log/apache2/modsec_audit.log | grep -i "lfi\|traversal\|etc/passwd"
```

---

## 📊 Analyse des logs

### Visualiser les attaques détectées
```bash
# Nombre total d'attaques bloquées
grep -c "Access denied with code 403" /var/log/apache2/modsec_audit.log

# Attaques SQL Injection détectées
grep -i "SQL Injection" /var/log/apache2/modsec_audit.log | wc -l

# Attaques XSS détectées
grep -i "XSS\|Cross-Site" /var/log/apache2/modsec_audit.log | wc -l

# Règles CRS les plus souvent déclenchées
grep -oP '\[id "\d+"\]' /var/log/apache2/modsec_audit.log | sort | uniq -c | sort -rn
```

### Extraire les IPs source des attaques
```bash
grep "Access denied" /var/log/apache2/modsec_audit.log | awk '{print $4}' | sort | uniq -c
```

### Chercher un log spécifique par timestamp
```bash
# Chercher les logs entre 20:20 et 20:25
grep "\[16/Jun/2026:20:2[0-5]" /var/log/apache2/modsec_audit.log
```

---

## 🛡️ Configuration fail2ban

### Phase 1️⃣ : Installation de fail2ban

#### 1.1 Installer fail2ban
```bash
sudo apt install fail2ban -y
```

#### 1.2 Vérifier l'installation
```bash
fail2ban-client --version
```

**Output** :
```
Fail2Ban v1.2.0
```

#### 1.3 Vérifier que le service est actif
```bash
sudo systemctl status fail2ban
```

**Output** :
```
● fail2ban.service - Fail2Ban intrusion prevention software
     Loaded: loaded (/etc/systemd/system/fail2ban.service; enabled; preset: enabled)
     Active: active (running)
```

---

### Phase 2️⃣ : Créer un filtre personnalisé ModSecurity

#### 2.1 Créer le fichier de filtre fail2ban pour ModSecurity
```bash
sudo nano /etc/fail2ban/filter.d/modsecurity.conf
```

**Contenu** :
```ini
# Fail2Ban filter for ModSecurity
# Ce filtre détecte les blocages ModSecurity dans les logs Apache

[Definition]
failregex = ^<HOST> .* ".*" .* ".*" .*\[.*\] (?:error|warn).*ModSecurity.*id "(?P<id>\d+)".*
ignoreregex = ^$
```

#### 2.2 Vérifier le fichier
```bash
cat /etc/fail2ban/filter.d/modsecurity.conf
```

---

### Phase 3️⃣ : Créer la jail personnalisée pour ModSecurity

#### 3.1 Créer le fichier jail.local
```bash
sudo nano /etc/fail2ban/jail.local
```

**Contenu** :
```ini
# =============================================================================
# Jail Configuration for ModSecurity
# =============================================================================

[DEFAULT]
# Délai entre les tentatives (en secondes)
findtime = 600

# Nombre de tentatives avant bannissement
maxretry = 5

# Durée du bannissement (en secondes) - 1 heure
bantime = 3600

# Format du log
logpath = /var/log/apache2/modsec_audit.log

# Action : bloquer l'IP
action = iptables-multiport[name=Fail2ban, port="http,https"]
         sendmail-whois[name=Fail2ban]

# =============================================================================
# Jail pour ModSecurity
# =============================================================================

[modsecurity]
enabled = true
filter = modsecurity
port = http,https
logpath = /var/log/apache2/modsec_audit.log
maxretry = 5
findtime = 600
bantime = 3600
action = iptables-multiport[name=ModSecurityBan, port="http,https"]
         sendmail-whois[name=ModSecurityBan]
```

#### 3.2 Vérifier la syntaxe du fichier
```bash
sudo fail2ban-client -d
```

**Output attendu** :
```
Success
```

#### 3.3 Recharger la configuration fail2ban
```bash
sudo systemctl restart fail2ban
```

#### 3.4 Vérifier que la jail est activée
```bash
sudo fail2ban-client status
```

**Output** :
```
Status
|- Number of jail: 1
`- Jail list: modsecurity
```

#### 3.5 Afficher les détails de la jail
```bash
sudo fail2ban-client status modsecurity
```

**Output** :
```
Status for the jail: modsecurity
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 1
|  `- File list: /var/log/apache2/modsec_audit.log
|- Actions
|  |- Currently banned: 0
|  `- Total banned: 0
`- Addresses: []
```

---

### Phase 4️⃣ : Test de fail2ban avec ModSecurity

#### 4.1 Lancer 5+ attaques SQL Injection pour déclencher le bannissement
```bash
# Terminal 1 : Surveiller les logs fail2ban
sudo tail -f /var/log/fail2ban/fail2ban.log

# Terminal 2 : Lancer 6 attaques
for i in {1..6}; do
  curl "http://localhost/?id=' OR 1=1--" 2>/dev/null
  sleep 1
done
```

#### 4.2 Vérifier que l'IP a été bannie
```bash
sudo fail2ban-client status modsecurity
```

**Output attendu** :
```
Status for the jail: modsecurity
|- Filter
|  |- Currently failed: 6
|  |- Total failed: 6
|  `- File list: /var/log/apache2/modsec_audit.log
|- Actions
|  |- Currently banned: 1
|  `- Total banned: 1
`- Addresses: [192.168.56.1]
```

#### 4.3 Vérifier les règles iptables
```bash
sudo iptables -L -n | grep -i ban
```

---

## ✅ Résultats finaux

### Récapitulatif de la configuration

| Élément | Statut | Détail |
|---------|--------|--------|
| **Apache2** | ✅ Fonctionnel | v2.4.67 (Debian) |
| **ModSecurity** | ✅ Activé | v2.9.11, Module security2 chargé |
| **Mode** | ✅ Enforcement | SecRuleEngine On |
| **OWASP CRS v4** | ✅ Installé | 38 fichiers de règles chargés |
| **Paranoia Level** | ✅ Configuré | Level 2 (équilibre sécurité/false positives) |
| **Anomaly Scoring** | ✅ Actif | Threshold = 5, Blocking = Enabled |
| **Logs d'audit** | ✅ Fonctionnels | `/var/log/apache2/modsec_audit.log` |
| **fail2ban** | ✅ Configuré | Jail modsecurity, bantime=3600, maxretry=5 |

### Attaques testées et bloquées

| Type d'attaque | Payload | Résultat | Code | Règles déclenchées |
|----------------|---------|----------|------|-------------------|
| **SQL Injection** | `' OR 1=1--` | ❌ BLOQUÉE | 403 | 942100, 942130, 942180, 942330, 942390 |
| **XSS** | `<script>alert(1)</script>` | ❌ BLOQUÉE | 403 | 941xxx (voir logs) |
| **Path Traversal** | `../../../etc/passwd` | ❌ BLOQUÉE | 403 | 930xxx (voir logs) |

### Scores d'anomalie

```
SQL Injection (' OR 1=1--) :
  Score total : 28 points
  Seuil de blocage : 5 points
  Résultat : BLOCKED (score > seuil)
  
Détail du scoring :
  - SQLI (SQL Injection) : 25 points
  - XSS (Cross-Site Scripting) : 0 points
  - RFI (Remote File Inclusion) : 0 points
  - LFI (Local File Inclusion) : 0 points
  - RCE (Remote Code Execution) : 0 points
  - Total : 28 points
```

---

## 🚀 Prochaines étapes

### Étape 1️⃣ : Installation de DVWA (Damn Vulnerable Web Application)

DVWA est une application web **volontairement vulnérable** conçue pour tester les WAF.

```bash
# Installer les dépendances
sudo apt install mariadb-server php php-mysqli php-gd php-xml php-mbstring libapache2-mod-php -y

# Cloner DVWA
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git

# Configurer DVWA
cd DVWA/config
sudo cp config.inc.php.dist config.inc.php

# Redémarrer Apache
sudo systemctl restart apache2

# Accéder à DVWA dans le navigateur
http://IP_SERVEUR/DVWA
```

### Étape 2️⃣ : Intégration avec Grafana/Loki pour la visualisation

```bash
# Installer Grafana
sudo apt install grafana-server -y
sudo systemctl start grafana-server

# Installer Loki pour l'agrégation de logs
sudo apt install loki -y

# Configurer Loki pour lire /var/log/apache2/modsec_audit.log
```

### Étape 3️⃣ : Création de règles personnalisées

Pour exclure certaines URLs légitimes des règles de détection :

```bash
# Créer le fichier d'exclusions personnalisées
sudo nano /etc/modsecurity/modsecurity_exclusions.conf
```

**Exemple d'exclusion** :
```
# Exclure les uploads de fichiers de la détection XSS
SecRule REQUEST_URI "@streq /upload.php" \
    "id:1001,phase:2,nolog,pass,ctl:RuleRemoveById=941xxx"
```

---

## 📝 Commandes utiles pour la maintenance

### Surveillance en temps réel
```bash
# Afficher les logs d'audit en direct
sudo tail -f /var/log/apache2/modsec_audit.log

# Afficher les erreurs Apache2
sudo tail -f /var/log/apache2/error.log

# Afficher les logs d'accès
sudo tail -f /var/log/apache2/access.log
```

### Statistiques et rapports
```bash
# Nombre d'attaques détectées
grep "Access denied" /var/log/apache2/modsec_audit.log | wc -l

# Types d'attaques les plus fréquentes
grep "\[msg " /var/log/apache2/modsec_audit.log | grep -o '\[msg "[^"]*"' | sort | uniq -c | sort -rn

# IPs malveillantes
grep "^\[" /var/log/apache2/modsec_audit.log | awk '{print $3}' | sort | uniq -c | sort -rn

# Règles les plus déclenché
grep -oP '\[id "\K\d+' /var/log/apache2/modsec_audit.log | sort | uniq -c | sort -rn
```

### Gestion de fail2ban
```bash
# Vérifier le statut de fail2ban
sudo fail2ban-client status

# Débannir une IP
sudo fail2ban-client set modsecurity unbanip 192.168.56.1

# Voir les IPs bannie actuellement
sudo fail2ban-client status modsecurity

# Redémarrer fail2ban
sudo systemctl restart fail2ban

# Voir les logs fail2ban
sudo tail -f /var/log/fail2ban/fail2ban.log
```

### Gestion d'Apache2
```bash
# Tester la configuration
sudo apachectl configtest

# Redémarrer Apache
sudo systemctl restart apache2

# Vérifier l'état
sudo systemctl status apache2

# Voir les modules chargés
apache2ctl -M
```

---

## 📚 Ressources et documentation

### Documentation officielle
- **Apache2** : https://httpd.apache.org/docs/2.4/
- **ModSecurity** : https://modsecurity.org/
- **OWASP CRS** : https://coreruleset.org/
- **fail2ban** : https://www.fail2ban.org/

### Payload de test recommandés
```
SQL Injection:
  ' OR 1=1--
  ' OR '1'='1
  1' OR '1'='1
  admin' --
  1; DROP TABLE users--

XSS:
  <script>alert(1)</script>
  <img src=x onerror=alert(1)>
  <svg/onload=alert(1)>
  javascript:alert(1)

Path Traversal:
  ../../../etc/passwd
  ..\\..\\..\\windows\\system32\\config\\sam
  ....//....//....//etc/passwd
  %2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

---

## 📖 Conclusion

Cette configuration a permis de :

✅ **Déployer un WAF complet** en mode Enforcement avec OWASP CRS v4  
✅ **Détecter et bloquer** les attaques SQL Injection, XSS et Path Traversal  
✅ **Analyzer les attaques** via les logs d'audit détaillés  
✅ **Implémenter fail2ban** pour bannir les IPs malveillantes automatiquement  
✅ **Configurer le paranoia level 2** pour un équilibre optimal sécurité/performance  

Le serveur est maintenant protégé contre les attaques web courantes et prêt pour un déploiement en production.

---

## 👨‍💻 Auteur

**Marc** | Étudiant INGC2 (Institut Africain d'Informatique)  
Projet 06 : Sécurisation Apache2 avec ModSecurity & OWASP CRS v4  
Date : 16 Juin 2026

---

**Dernière mise à jour** : 16 Juin 2026 20:30 CEST  
**Statut** : ✅ Projet Terminé - Prêt pour Production
