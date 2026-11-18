# Guide de mise en place d'un Load Balancer avec HAProxy sous WSL

Ce guide récapitule les étapes complètes pour mettre en œuvre un projet de **clustering et de répartition de charge (Load Balancing)** utilisant **HAProxy** sous **WSL (Windows Subsystem for Linux)** avec l'algorithme d'alternance (**Round Robin**).

---

## Partie 1 : Installation de WSL

1. Ouvrir Windows PowerShell et « Exécuter en tant qu'administrateur ».
2. Lancer la commande d'installation automatique
   ```powershell
   wsl --install
   ```
3. Redémarrer votre PC
4. Configurer votre compte Linux
   - Cherchez « Ubuntu » dans le menu Démarrer et lancez-le
   - Il vous sera demandé de créer un nom d'utilisateur UNIX (entrez le nom de votre choix, par exemple votre prénom en minuscules, puis validez par Entrée).
5. `C'est terminé !`

## Partie 2 : Installation de HAProxy sous WSL

1. Ouvrez votre terminal **WSL (Ubuntu)**.
2. Mettez à jour la liste des paquets :
   ```bash
   sudo apt update
   ```
3. Installez HAProxy :
   ```bash
   sudo apt install haproxy -y
   ```
4. Vérifiez l'installation :
   ```bash
   haproxy -v
   ```

---

## Partie 3 : Configuration en local (Sur 1 seul PC)

L'objectif est de simuler 3 serveurs web sur une même machine en utilisant 3 ports différents (`8001`, `8002`, `8003`), répartis par un Load Balancer écoutant sur le port `8080`.

### 1. Création de l'arborescence et des fichiers `a.html`
Deja creer dans le zip

### 2. Lancement des 3 serveurs HTTP
Ouvrez **3 fenêtres de terminal WSL** dans le dossier ou se trouve les dossiers `Back` et lancez un serveur Python léger dans chacune :

* **Terminal 1 :**
  ```bash
  cd Back1 && python3 -m http.server 8001
  ```
* **Terminal 2 :**
  ```bash
  cd Back2 && python3 -m http.server 8002
  ```
* **Terminal 3 :**
  ```bash
  cd Back3 && python3 -m http.server 8003
  ```

### 3. Configuration de HAProxy
1. Ouvrez le fichier de configuration de HAProxy dans un 4ème terminal :
   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```
2. Ajoutez le bloc suivant tout en bas du fichier :
   ```haproxy
   frontend mon_load_balancer
      bind *:8080
      mode http
      default_backend mes_serveurs_a

   backend mes_serveurs_a
      mode http
      balance roundrobin
      option httpclose
      server srv1 127.0.0.1:8001 check
      server srv2 127.0.0.1:8002 check
      server srv3 127.0.0.1:8003 check
   ```
   > **Explication des directives :**
   > - `balance roundrobin` : Applique l'algorithme d'alternance stricte.
   > - `option httpclose` : Empêche le navigateur de garder une connexion ouverte (Keep-Alive) qui bloquerait l'alternance.
   > - `check` : Vérifie que le serveur est actif.

3. Enregistrez (`Ctrl + O`, `Entrée`) et quittez (`Ctrl + X`).

### 4. Vérification et démarrage
1. Testez la syntaxe du fichier :
   ```bash
   sudo haproxy -c -f /etc/haproxy/haproxy.cfg
   ```
2. Redémarrez le service :
   ```bash
   sudo service haproxy restart
   ```

### 5. Test de fonctionnement
* **Via le terminal :** Exécutez `curl http://localhost:8080/a.html` plusieurs fois.
* **Via le navigateur Windows :** Ouvrez `http://localhost:8080/a.html` et actualisez sans cache avec **`Ctrl + Shift + R`** pour observer l'alternance entre les 3 serveurs.

---

## Partie 4 : Passage à une architecture distribuée (Sur 3 PC)

Dans cette configuration, les 3 PC sont connectés au même réseau local (Wi-Fi ou Ethernet).

**Exemple d'adressage IP du réseau local** (à vérifier via `ipconfig` sous Windows) :
* **PC 1 (Load Balancer + Serveur 1) :** `192.168.1.10`
* **PC 2 (Serveur 2) :** `192.168.1.11`
* **PC 3 (Serveur 3) :** `192.168.1.12`

### 1. Lancement des serveurs sur chaque PC
Sur **chacun des 3 PC**, démarrez le serveur web Python dans le dossier contenant son fichier `a.html` (port `8001` par exemple) :

```bash
cd Back1 && python3 -m http.server 8001
```

### 2. Ouverture du Pare-feu Windows (Sur chaque PC)
Pour permettre au Load Balancer de communiquer avec les autres PC via le réseau, ouvrez **PowerShell en tant qu'administrateur** sur les PC hébergeant les serveurs et exécutez :

```powershell
New-NetFirewallRule -DisplayName "Autoriser Serveur Python Cluster" -Direction Inbound -Protocol TCP -LocalPort 8001 -Action Allow
```
*(Si le Load Balancer doit aussi être accessible depuis d'autres PC, exécutez la même commande sur le PC 1 en remplaçant `8001` par `8080`).*

### 3. Mise à jour de HAProxy (Uniquement sur le PC 1 - Load Balancer)
Modifiez le fichier `/etc/haproxy/haproxy.cfg` sur le PC principal pour remplacer `127.0.0.1` par les adresses IP réelles des 3 machines :

```haproxy
frontend mon_load_balancer
    bind *:8080
    mode http
    default_backend mes_serveurs_a

backend mes_serveurs_a
    mode http
    balance roundrobin
    option httpclose
    server srv1 192.168.1.10:8001 check
    server srv2 192.168.1.11:8001 check
    server srv3 192.168.1.12:8001 check
```

Redémarrez ensuite HAProxy :
```bash
sudo service haproxy restart
```

### 4. Test final sur le réseau
Depuis n'importe quel PC connecté au réseau, ouvrez un navigateur et accédez à l'adresse IP du Load Balancer :
👉 **`http://192.168.1.10:8080/a.html`**

En actualisant la page (`Ctrl + Shift + R`), le trafic est distribué en alternance entre les 3 ordinateurs.