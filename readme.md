# Livrables — TP Bypass SSL Pinning

Salma AIT ZIDAN
---

## Livrable 1 — Capture PowerShell

```powershell
PS C:\Users\HP\Downloads> objection version
objection: 1.12.5
PS C:\Users\HP> frida --version
16.6.6
PS C:\Users\HP> python -c "import frida; print(frida.__version__)"
16.6.6
PS C:\Users\HP> frida-ps -Uai
  PID  Name                  Identifier
-----  --------------------  ---------------------------------------
13547  Chrome                com.android.chrome
13064  Clock                 com.google.android.deskclock
 1868  Google                com.google.android.googlequicksearchbox
```

---

## Livrable 2 — Commande exacte utilisée

```powershell
objection -g com.example.projetws explore --startup-command "android sslpinning disable"
```

> Package cible : `com.example.projetws`

---

## Livrable 3 — Capture mitmproxy

Capture de **mitmweb** (`http://127.0.0.1:8081`) montrant une requête HTTPS de `com.example.projetws` interceptée via `192.168.11.125:8080`.
<img width="856" height="463" alt="image" src="https://github.com/user-attachments/assets/5e7e6689-490d-4326-ba28-d84ad9f266ad" />
