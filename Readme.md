<img width="990" height="521" alt="Screenshot 2026-04-27 094452" src="https://github.com/user-attachments/assets/4a8d641a-f52b-4efb-8e34-276330b9ccf0" /># Lab Frida — Rapport d'analyse de sécurité mobile 
**Appareil :** Android Emulator 5554  
**Version Frida :** 17.9.1  
**Date :** 2026-04-27

---

## Exercice 1 — Installation et preuve

### 1.1 Version Frida CLI
```
PS C:\Users\hp> frida --version
17.9.1
```

### 1.2 Version bibliothèque Python
```
PS C:\Users\hp> python -c "import frida; print(frida.__version__)"
17.9.1
```
<img width="1101" height="258" alt="Screenshot 2026-04-27 092339" src="https://github.com/user-attachments/assets/0e18e429-590c-4be7-a203-575ce96e0cc6" />

### 1.3 Appareils ADB
```
PS C:\Users\hp> adb devices
List of devices attached
emulator-5554   device
```

---

## Exercice 2 — Déploiement Android

### 2.1 Architecture détectée
```
adb shell getprop ro.product.cpu.abi
x86_64
```
Fichier téléchargé : `frida-server-17.9.1-android-x86_64`

### 2.2 Déploiement de frida-server

**Copie vers l'émulateur**
```bash
adb push "D:\S2\programmation mobile\frida-server-17.9.1-android-x86_64\frida-server-17.9.1-android-x86_64" /data/local/tmp/frida-server
```
Résultat : `110787848 bytes in 1.767s`

<img width="1722" height="103" alt="Screenshot 2026-04-27 093601" src="https://github.com/user-attachments/assets/0b2bac3a-f579-4200-8cf7-010e64f81f52" />

**Permissions d'exécution**
```bash
adb shell chmod 755 /data/local/tmp/frida-server
```

**Lancement en root**
```bash
adb root
adb shell "/data/local/tmp/frida-server &"
```



**Vérification**
```bash
adb shell ps | Select-String "frida"
```
Résultat : `shell  5426  1  15396660  43604  do_sys_poll  0 S frida-server`

<img width="1275" height="284" alt="Screenshot 2026-04-27 093811" src="https://github.com/user-attachments/assets/9b9e7540-54ef-4b93-8298-b7e1e76600f2" />

**Redirection de ports**
```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

<img width="783" height="228" alt="Screenshot 2026-04-27 093859" src="https://github.com/user-attachments/assets/9b74c27e-21e3-4c80-bfb0-21cfad46efda" />

### 2.3 Liste des applications (frida-ps -Uai)
```
PID   Name                  Identifier
----  --------------------  -----------------------------------------
4462  Google                com.google.android.googlequicksearchbox
5352  Phone                 com.google.android.dialer
5441  Photos                com.google.android.apps.photos
   -  JNIDemo               com.example.jnidemo
   -  Uncrackable Level 2   owasp.mstg.uncrackable2
   -  Uncrackable1          owasp.mstg.uncrackable1
```
<img width="1195" height="741" alt="Screenshot 2026-04-27 094002" src="https://github.com/user-attachments/assets/39bfc35c-1b3d-4d68-9840-7a1d2489e162" />

---

## Exercice 3 — Injection minimale

### 3.1 Script hello.js — Test Java
```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");
});
```

**Commande :**
```bash
frida -U -n "JNIDemo" -l C:\Users\hp\hello.js
```

**Résultat :**
```
Attaching...
[+] Frida Java.perform OK
[Android Emulator 5554::JNIDemo ]->
```

**Interprétation :** Frida est correctement connecté à l'émulateur, le processus JNIDemo est instrumenté et l'API Java fonctionne.

### 3.2 Script hello_native.js — Test hook natif
```javascript
console.log("[+] Script chargé");

Interceptor.attach(Module.getExportByName(null, "recv"), {
  onEnter(args) {
    console.log("[+] recv appelée");
  }
});
```

**Commande :**
```bash
frida -U -n "JNIDemo" -l C:\Users\hp\hello_native.js
```

**Résultat :**
```
[+] Script chargé
```

**Interprétation :** Le hooking natif fonctionne. `recv` sera interceptée dès qu'une opération réseau sera déclenchée.

---

## Exercice 4 — Étape 6 : Console interactive Frida

### Commandes exécutées et résultats

**Architecture du processus :**
```javascript
Process.arch
// => "x64"
```

**Identifiant du processus :**
```javascript
Process.id
// => 5618
```

**Module principal :**
```javascript
Process.mainModule
// => {
//   "base": "0x5ec284288000",
//   "name": "app_process64",
//   "path": "/system/bin/app_process64",
//   "size": 65536
// }
```
<img width="990" height="521" alt="Screenshot 2026-04-27 094452" src="https://github.com/user-attachments/assets/9409bcb3-0818-4b9a-94bb-0f1ec5ce437d" />

**Informations sur libc.so :**
```javascript
Process.getModuleByName("libc.so")
// => {
//   "base": "0x755e0784c000",
//   "name": "libc.so",
//   "path": "/apex/com.android.runtime/lib64/bionic/libc.so",
//   "size": 1212416
// }
```

**Adresse de la fonction recv :**
```javascript
Process.getModuleByName("libc.so").getExportByName("recv")
// => "0x755e078aaf00"
```
<img width="1654" height="581" alt="Screenshot 2026-04-27 094640" src="https://github.com/user-attachments/assets/37f20d06-4815-413d-97ef-68b2fb82f54b" />

**Interprétation :** La fonction `recv` est localisée en mémoire et peut être interceptée. Cela confirme que les appels réseau natifs de l'application sont observables.

**Vérification Java :**
```javascript
Java.available
// => true
```

**Bibliothèques de chiffrement détectées :**
```javascript
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1 ||
  m.name.indexOf("boring") !== -1
)
// => [
//   { "name": "libcrypto.so",          "path": "/system/lib64/libcrypto.so" },
//   { "name": "libjavacrypto.so",       "path": "/apex/com.android.conscrypt/lib64/" },
//   { "name": "libcrypto.so",           "path": "/apex/com.android.conscrypt/lib64/" },
//   { "name": "libssl.so",              "path": "/apex/com.android.conscrypt/lib64/" },
//   { "name": "libcrypto_httpengine.so","path": "/apex/com.android.tethering/lib64/" }
// ]
```

**Interprétation :** L'application utilise des composants natifs SSL/TLS via Conscrypt (implémentation Android de BoringSSL). Les communications réseau sécurisées sont donc présentes.

**Bibliothèque native de l'application :**
```
libnative-lib.so => /data/app/.../com.example.jnidemo.../base.apk!/lib/x86_64/libnative-lib.so
```

**Interprétation :** La bibliothèque JNI propre à JNIDemo est chargée depuis l'APK. C'est le code C++ natif qui effectue les calculs et vérifications affichés dans l'app.

---

## Exercice 5 — Étape 7 : Hooks natifs réseau et fichiers

### 5.1 hook_connect.js
```javascript
console.log("[+] Hook connect chargé");

const connectPtr = Process.getModuleByName("libc.so").getExportByName("connect");
console.log("[+] connect trouvée à : " + connectPtr);

Interceptor.attach(connectPtr, {
  onEnter(args) {
    console.log("[+] connect appelée");
    console.log("    fd = " + args[0]);
    console.log("    sockaddr = " + args[1]);
  },
  onLeave(retval) {
    console.log("    retour = " + retval.toInt32());
  }
});
```

### 5.2 hook_network.js
```javascript
console.log("[+] Hooks réseau chargés");

const sendPtr = Process.getModuleByName("libc.so").getExportByName("send");
const recvPtr = Process.getModuleByName("libc.so").getExportByName("recv");

console.log("[+] send trouvée à : " + sendPtr);
console.log("[+] recv trouvée à : " + recvPtr);

Interceptor.attach(sendPtr, {
  onEnter(args) {
    console.log("[+] send appelée");
    console.log("    fd = " + args[0]);
    console.log("    len = " + args[2].toInt32());
  }
});

Interceptor.attach(recvPtr, {
  onEnter(args) {
    console.log("[+] recv appelée");
    console.log("    fd = " + args[0]);
    console.log("    len demandé = " + args[2].toInt32());
  },
  onLeave(retval) {
    console.log("    recv retourne = " + retval.toInt32());
  }
});
```

**Résultat observé :**
```
[+] Hooks réseau chargés
[+] send trouvée à : 0x755e078dda80
[+] recv trouvée à : 0x755e078aaf00
```

**Interprétation :** Les fonctions `send` et `recv` sont localisées et hookées. Aucun appel réseau natif n'a été déclenché par JNIDemo durant le test, ce qui indique que l'app n'effectue pas de communication réseau active via ces fonctions bas niveau.

### 5.3 hook_file.js
```javascript
console.log("[+] Hook fichiers chargé");

const openPtr = Process.getModuleByName("libc.so").getExportByName("open");
const readPtr = Process.getModuleByName("libc.so").getExportByName("read");

console.log("[+] open trouvée à : " + openPtr);
console.log("[+] read trouvée à : " + readPtr);

Interceptor.attach(openPtr, {
  onEnter(args) {
    this.path = args[0].readUtf8String();
    console.log("[+] open appelée : " + this.path);
  }
});

Interceptor.attach(readPtr, {
  onEnter(args) {
    console.log("[+] read appelée — fd=" + args[0] + " taille=" + args[2].toInt32());
  }
});
```

**Résultat observé :**
```
[+] Hook fichiers chargé
[+] open trouvée à : 0x755e078a79d0
[+] read trouvée à : 0x755e078a9740
[+] read appelée — fd=0x5c taille=8
[+] read appelée — fd=0x65 taille=524288
```

**Interprétation :** L'application effectue des lectures de fichiers en arrière-plan. La lecture volumineuse (524288 octets) sur fd=0x65 correspond probablement à `/proc/self/maps`, fichier utilisé par l'app pour analyser sa propre mémoire — visible dans l'interface de l'app (`JNI · NDK · ptrace : /proc/self/maps`).

---

## Exercice 6 — Étape 8 : Hooks Java

### 6.1 hook_prefs.js
```javascript
Java.perform(function () {
  console.log("[+] Hook SharedPreferences chargé");

  var Impl = Java.use("android.app.SharedPreferencesImpl");

  Impl.getString.overload("java.lang.String", "java.lang.String").implementation = function (key, defValue) {
    var result = this.getString(key, defValue);
    console.log("[SharedPreferences][getString] key=" + key + " => " + result);
    return result;
  };

  Impl.getBoolean.overload("java.lang.String", "boolean").implementation = function (key, defValue) {
    var result = this.getBoolean(key, defValue);
    console.log("[SharedPreferences][getBoolean] key=" + key + " => " + result);
    return result;
  };
});
```

**Résultat :** Hook chargé avec succès sur Uncrackable1. Aucune lecture de SharedPreferences détectée — l'application ne stocke pas de données de configuration via ce mécanisme.

### 6.2 hook_debug.js
```javascript
Java.perform(function () {
  console.log("[+] Hook Debug chargé");

  var Debug = Java.use("android.os.Debug");

  Debug.isDebuggerConnected.implementation = function () {
    var result = this.isDebuggerConnected();
    console.log("[Debug] isDebuggerConnected() => " + result);
    return result;
  };

  Debug.waitingForDebugger.implementation = function () {
    var result = this.waitingForDebugger();
    console.log("[Debug] waitingForDebugger() => " + result);
    return result;
  };
});
```

**Résultat :** Hook chargé. Uncrackable1 utilise `System.exit()` directement plutôt que `Debug.isDebuggerConnected()` pour ses vérifications de sécurité.

### 6.3 hook_runtime.js
```javascript
Java.perform(function () {
  console.log("[+] Hook Runtime.exec chargé");

  var Runtime = Java.use("java.lang.Runtime");

  Runtime.exec.overload("java.lang.String").implementation = function (cmd) {
    console.log("[Runtime.exec] " + cmd);
    return this.exec(cmd);
  };

  var System = Java.use("java.lang.System");
  System.exit.implementation = function (code) {
    console.log("[System.exit] appelé avec code=" + code + " — bloqué !");
  };
});
```

---

## Exercice 7 — Dépannage documenté

### Problème rencontré
```
Failed to attach: unable to access process with pid 5618
```

### Diagnostic
Frida n'avait pas les permissions suffisantes pour s'attacher au processus.

### Solution appliquée
```bash
adb root
adb shell "/data/local/tmp/frida-server &"
frida -U -n "JNIDemo" -l C:\Users\hp\hello.js
```

Le redémarrage de frida-server en mode root a résolu le problème.

---

## Observations de sécurité — JNIDemo

| Élément | Observation |
|---|---|
| Architecture | x64 (émulateur) |
| PID | 5618 |
| Bibliothèque native | libnative-lib.so chargée depuis l'APK |
| SSL/TLS | libssl.so + libcrypto.so via Conscrypt |
| Détection émulateur | Présente mais contournable (affiche "NON") |
| Accès fichiers | Lecture de /proc/self/maps détectée |
| Activité réseau | Aucune via send/recv natif |
| Runtime Java | Disponible (Java.available = true) |

---

## Conclusion

Ce lab a permis de maîtriser les étapes fondamentales de l'analyse dynamique avec Frida :

1. **Installation** du client Frida et vérification des versions.
2. **Déploiement** de frida-server sur un émulateur Android x86_64.
3. **Connexion** et validation via frida-ps.
4. **Injection** de scripts JavaScript dans un processus Android.
5. **Exploration** de la console interactive pour identifier les composants natifs.
6. **Hooking natif** des fonctions réseau (send, recv, connect) et fichiers (open, read).
7. **Hooking Java** des classes SharedPreferences, Debug et Runtime.
8. **Identification** des bibliothèques de chiffrement SSL/TLS chargées dans le processus.
