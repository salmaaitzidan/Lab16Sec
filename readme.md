# TP — Bypass SSL Pinning avec Objection & Frida
### Analyse de trafic HTTPS — `com.example.projetws`
**Environnement :** Windows 11 · Python 3.11.0 · mitmproxy · Émulateur Pixel 6

---

## Objectifs du TP

À l'issue de ce TP, tu seras capable de :

- Installer et configurer Objection (surcouche Frida) sur ta machine Windows
- Mettre en place mitmproxy et installer sa CA sur l'émulateur Pixel 6
- Désactiver le SSL pinning de `com.example.projetws` via `android sslpinning disable`
- Capturer et analyser le trafic HTTPS en clair dans mitmproxy

---

## Prérequis

- Python 3.11.0 déjà installé ✓
- ADB fonctionnel (`adb devices` retourne l'émulateur)
- Émulateur Pixel 6 lancé via Android Studio (AVD)
- mitmproxy installé
- **Ton IP Wi-Fi machine : `192.168.11.125`** (à utiliser partout dans ce TP)

---

## Étape 1 — Installer Objection et Frida

Ouvre **PowerShell** (`Win+R` → `powershell`) et lance :

### Option recommandée (environnement isolé avec pipx)

```powershell
pip install --user pipx
pipx ensurepath
pipx install objection
```

Ferme et rouvre PowerShell après `pipx ensurepath` pour recharger le PATH.

### Option alternative (pip classique)

```powershell
pip install --upgrade objection frida frida-tools
```

### Vérification

```powershell
objection --version
frida --version
python -c "import frida; print(frida.__version__)"
```

Les trois commandes doivent retourner un numéro de version sans erreur.

> **Si `objection` n'est pas reconnu :** ajoute ce dossier au PATH Windows :
> `C:\Users\HP\AppData\Roaming\Python\Python311\Scripts`
> (Paramètres système > Variables d'environnement > Path > Nouveau)

---

## Étape 2 — Préparer l'émulateur et démarrer frida-server

### 2.1 — Lancer l'émulateur Pixel 6

Démarre l'AVD Pixel 6 depuis Android Studio, puis vérifie qu'il est visible :

```powershell
adb devices
# Attendu : emulator-5554   device
```

### 2.2 — Identifier l'architecture CPU

```powershell
adb shell getprop ro.product.cpu.abi
# Pour un émulateur Pixel 6 sur x86_64 machine : x86_64
```

### 2.3 — Télécharger frida-server

Rends-toi sur [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases) et télécharge :

```
frida-server-<VERSION>-android-x86_64.xz
```

La `<VERSION>` doit correspondre **exactement** à ce que retourne `frida --version`.

Décompresse avec **7-Zip** → tu obtiens un fichier sans extension, renomme-le `frida-server`.

### 2.4 — Pousser et lancer frida-server

```powershell
adb push C:\Users\HP\Downloads\frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

Laisse ce terminal ouvert — `frida-server` doit rester actif.

### 2.5 — Forward des ports (recommandé sous Windows)

Dans un **nouveau** terminal PowerShell :

```powershell
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

### 2.6 — Vérifier la détection de l'émulateur

```powershell
frida-ps -Uai
# Doit lister les apps installées sur l'émulateur
```

---

## Étape 3 — Configurer mitmproxy et installer la CA

### 3.1 — Démarrer mitmproxy

```powershell
mitmproxy --listen-host 192.168.11.125 --listen-port 8080
```

Ou en mode web (plus lisible) :

```powershell
mitmweb --listen-host 192.168.11.125 --listen-port 8080
```

Ton proxy écoute sur : **`192.168.11.125:8080`**

### 3.2 — Configurer le proxy sur l'émulateur

Dans l'émulateur Pixel 6 :
**Paramètres > Réseau et Internet > Wi-Fi > AndroidWifi (maintenir appuyé) > Modifier > Options avancées > Proxy manuel**

- **Nom d'hôte du proxy :** `192.168.11.125`
- **Port :** `8080`

### 3.3 — Installer la CA mitmproxy

Depuis le navigateur de l'émulateur, visite :

```
http://mitm.it
```

Clique sur **Android** → télécharge et installe le certificat.

> Sur un émulateur AVD standard (non-rooted), installe le certificat en CA utilisateur : **Paramètres > Sécurité > Chiffrement et identifiants > Installer un certificat > Certificat CA**

### Validation rapide

Ouvre Chrome sur l'émulateur, visite `https://example.com`. Les requêtes doivent apparaître dans mitmproxy. Si c'est le cas, tu es prêt pour l'étape Objection.

---

## Étape 4 — Lancer `com.example.projetws` avec Objection

### 4.1 — Confirmer le package

```powershell
frida-ps -Uai | Select-String "projetws"
# Attendu : com.example.projetws
```

### 4.2 — Stratégie 1 : Spawn (recommandée — patch au démarrage)

```powershell
objection -g com.example.projetws explore --startup-command "android sslpinning disable"
```

### 4.3 — Stratégie 2 : Attach (si le spawn crashe)

```powershell
# 1. Ouvre com.example.projetws manuellement sur l'émulateur
# 2. Attache Objection :
objection -g com.example.projetws explore

# 3. Dans la console Objection :
android sslpinning disable
```

### Résultat attendu

La console Objection affiche des lignes confirmant l'installation des hooks SSL. L'app tourne normalement et les requêtes HTTPS apparaissent dans mitmproxy.

### Commandes utiles dans la console Objection

```
help android sslpinning
android hooking search classes pin
android hooking list classes
```

---

## Étape 5 — Validation

1. Navigue dans `com.example.projetws` (login, appels API, etc.) pour générer du trafic.
2. Dans **mitmweb** (`http://127.0.0.1:8081`) ou la console mitmproxy, vérifie que :
   - Les requêtes HTTPS de l'app apparaissent avec leur URL complète
   - Les corps de réponse sont lisibles sans erreur SSL côté app
3. Dans la console Objection, observe les hooks déclenchés à chaque appel réseau.

---

## Livrables attendus

| # | Livrable |
|---|---|
| 1 | Capture PowerShell : `objection --version`, `frida --version`, `frida-ps -Uai` |
| 2 | Commande exacte utilisée (spawn ou attach) |
| 3 | Capture mitmproxy/mitmweb montrant une requête HTTPS de `com.example.projetws` |

---

## Commande tout-en-une (si spawn stable)

```powershell
objection -g com.example.projetws explore --startup-command "android sslpinning disable"
```

---

## Dépannage courant

| Problème | Cause probable | Solution |
|---|---|---|
| `frida-ps` ne voit pas l'émulateur | Versions frida/frida-server différentes | Aligne les versions exactement |
| `objection` non reconnu dans PowerShell | Scripts Python311 absent du PATH | Ajoute `C:\Users\HP\AppData\Roaming\Python\Python311\Scripts` au PATH |
| L'app crashe au spawn | Pinning vérifié très tôt ou anti-Frida | Bascule en attach |
| mitmproxy ne voit rien | Proxy émulateur mal configuré ou CA absente | Vérifie IP `192.168.11.125:8080` + CA installée |
| Émulateur inaccessible via ADB | AVD non démarré | Lance l'AVD depuis Android Studio avant `adb devices` |

---

## Checklist finale

- [ ] `objection`, `frida`, `frida-server` installés et versions alignées
- [ ] mitmproxy actif sur `192.168.11.125:8080` + CA installée sur l'émulateur
- [ ] Proxy Wi-Fi configuré sur l'émulateur Pixel 6
- [ ] Objection lancé sur `com.example.projetws` avec `android sslpinning disable`
- [ ] Trafic HTTPS de l'app visible dans mitmproxy
- [ ] Nettoyage post-test : stopper `frida-server`, retirer le proxy de l'émulateur

---

*TP réalisé sur Windows 11 — Python 3.11.0 — Émulateur Pixel 6 (AVD) — mitmproxy*
*Usage pédagogique uniquement. Ne jamais utiliser ces techniques sans autorisation explicite.*
