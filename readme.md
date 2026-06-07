# TP — Bypass SSL Pinning avec Objection & Frida
### Analyse de trafic HTTPS d'applications Android

---

## Objectifs du TP

À l'issue de ce TP, tu seras capable de :

- Installer et configurer Objection (surcouche Frida) sur ta machine
- Mettre en place un proxy d'interception (Burp Suite ou mitmproxy) avec sa CA
- Désactiver le SSL pinning d'une app Android cible via `android sslpinning disable`
- Capturer et analyser le trafic HTTPS en clair dans le proxy

---

## Prérequis

- Python 3.x installé sur ta machine (vérifie avec `python --version`)
- ADB fonctionnel (`adb devices` retourne ton appareil)
- Un appareil Android rooté ou un émulateur (ex. : Genymotion, AVD avec accès root)
- Burp Suite Community ou mitmproxy installé
- Connexion Wi-Fi partagée entre la machine et l'appareil

---

## Étape 1 — Installer Objection et Frida sur ta machine

### Option recommandée (environnement isolé avec pipx)

```bash
pip install --user pipx
pipx ensurepath
pipx install objection
```

### Option alternative (pip classique)

```bash
pip install --upgrade objection frida frida-tools
```

### Vérification de l'installation

```bash
objection --version
frida --version
python -c "import frida; print(frida.__version__)"
```

Les trois commandes doivent retourner un numéro de version sans erreur.

> **Note Windows :** si `objection` n'est pas reconnu après installation, ajoute manuellement le dossier `Scripts` de Python au PATH :
> `%USERPROFILE%\AppData\Roaming\Python\Python3xx\Scripts`

---

## Étape 2 — Préparer l'appareil et démarrer frida-server

### 2.1 — Activer le débogage USB

Sur l'appareil : **Paramètres > Options développeur > Débogage USB**. Branche le câble USB et accepte l'empreinte sur l'appareil.

```bash
adb devices
# Doit afficher ton appareil avec le statut "device"
```

### 2.2 — Identifier l'architecture CPU

```bash
adb shell getprop ro.product.cpu.abi
# Exemple de sortie : arm64-v8a
```

Note le résultat — tu en as besoin pour télécharger le bon binaire.

### 2.3 — Télécharger frida-server

Rends-toi sur [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases) et télécharge :

```
frida-server-<VERSION>-android-<ARCH>.xz
```

Exemple : `frida-server-16.2.1-android-arm64.xz`

**Important :** la version de `frida-server` doit correspondre exactement à celle de `frida` installée sur ta machine (`frida --version`).

Décompresse l'archive (7-Zip sous Windows, `unxz` sous Linux/macOS).

### 2.4 — Pousser et lancer frida-server sur l'appareil

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

Laisse ce terminal ouvert — `frida-server` doit rester en cours d'exécution.

### 2.5 — (Optionnel) Forward des ports

Si la connexion est instable :

```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

### 2.6 — Vérifier la détection de l'appareil

Dans un nouveau terminal :

```bash
frida-ps -Uai
# Doit lister les apps installées sur l'appareil
```

---

## Étape 3 — Configurer le proxy et installer la CA

### 3.1 — Démarrer le proxy

Lance **Burp Suite** (ou mitmproxy) sur ta machine et note l'adresse IP et le port d'écoute.

Exemple : `192.168.1.42:8080`

### 3.2 — Configurer le proxy Wi-Fi sur l'appareil

Sur l'appareil : **Paramètres > Wi-Fi > (maintenir appuyé sur le réseau) > Modifier > Proxy manuel**

- Hôte : ton IP machine (ex. `192.168.1.42`)
- Port : `8080`

### 3.3 — Installer la CA du proxy

**Avec Burp Suite :**
Depuis le navigateur de l'appareil, visite `http://burp` → télécharge et installe le certificat.

**Avec mitmproxy :**
Visite `http://mitm.it` → section Android → installe le certificat.

### Validation rapide

Ouvre un navigateur sur l'appareil et visite un site HTTPS quelconque. Les requêtes doivent apparaître dans le proxy. L'app cible peut encore refuser — c'est normal, l'étape Objection vient ensuite.

> **Rappel :** l'SSL pinning désactivé ne remplace pas la CA. Le patch Objection neutralise la vérification côté app (OkHttp, TrustManagers), mais tu as quand même besoin de la CA pour un MITM propre sans erreurs.

---

## Étape 4 — Lancer l'app avec Objection et désactiver le pinning

### 4.1 — Identifier le package de l'app cible

```bash
frida-ps -Uai | grep -i <nom_app>
# Exemple : frida-ps -Uai | grep -i bank
```

Note le nom complet du package, ex. : `com.example.bankapp`

### 4.2 — Stratégie 1 : Injection au démarrage (spawn) — recommandée

À utiliser quand l'app effectue la vérification du pinning très tôt au lancement :

```bash
objection -g com.example.bankapp explore --startup-command "android sslpinning disable"
```

### 4.3 — Stratégie 2 : Attachement à une app en cours (attach)

À utiliser si le spawn provoque un crash :

```bash
# 1. Ouvre l'app manuellement sur l'appareil
# 2. Attache Objection :
objection -g com.example.bankapp explore

# 3. Dans la console Objection qui s'ouvre :
android sslpinning disable
```

### Résultat attendu

Dans la console Objection, tu dois voir des messages confirmant l'installation des hooks SSL. L'app ne présente plus d'erreur de certificat et les requêtes HTTPS apparaissent dans le proxy.

### Commandes utiles dans la console Objection

```
help android sslpinning
android hooking search classes pin
android hooking list classes
```

---

## Étape 5 — Validation

1. Utilise les fonctionnalités de l'app (login, écrans API) pour générer du trafic.
2. Dans **Burp/mitmproxy**, vérifie que :
   - Les requêtes HTTPS de l'app apparaissent bien
   - Les réponses sont visibles sans alerte SSL côté app
3. Dans la console **Objection**, observe les hooks déclenchés.

---

## Livrables attendus

| Livrable | Description |
|---|---|
| Capture 1 | Sortie de `objection --version`, `frida --version`, `frida-ps -Uai` |
| Capture 2 | Commande exacte utilisée (`--startup-command` ou `attach`) |
| Capture 3 | Proxy montrant une requête HTTPS de l'app cible interceptée |

---

## Bonnes pratiques

- Commence toujours en **spawn** pour patcher dès le départ ; bascule en **attach** si instable.
- N'empile pas plusieurs commandes dans `--startup-command` (une à la fois pour isoler les problèmes).
- Minimise les logs en session réelle — certaines apps détectent un IO verbeux.
- **Nettoie après le test :** stoppe `frida-server`, restaure les paramètres réseau, supprime la CA du téléphone.

---

## Variante tout-en-une

Si l'app tolère le spawn, cette commande suffit souvent :

```bash
objection -g com.example.app explore --startup-command "android sslpinning disable"
```

Navigue ensuite dans l'app — le trafic doit apparaître en clair dans le proxy.

---

## Annexe — Dépannage courant

| Problème | Cause probable | Solution |
|---|---|---|
| `frida-ps` ne voit pas l'appareil | Version frida/frida-server différentes | Aligne les versions exactement |
| L'app crashe au spawn | Vérification de pinning trop précoce ou protection anti-frida | Essaie attach ; sinon script Frida natif |
| Le proxy ne voit rien | CA non installée ou proxy mal configuré | Vérifie config Wi-Fi + CA installée |
| App utilise Cronet/natif | OkHttp non utilisé, layer natif | Script Frida bas niveau nécessaire |

---

## Checklist finale

- [ ] `objection`, `frida`, `frida-server` installés et versions alignées
- [ ] Proxy opérationnel (Burp ou mitmproxy) + CA installée sur l'appareil
- [ ] Lancement Objection (spawn ou attach) avec `android sslpinning disable` exécuté
- [ ] Trafic HTTPS de l'app visible dans le proxy en clair
- [ ] Nettoyage post-test effectué

---

*TP rédigé pour un usage pédagogique en environnement de test autorisé. Ne jamais utiliser ces techniques sur des applications sans autorisation explicite.*
