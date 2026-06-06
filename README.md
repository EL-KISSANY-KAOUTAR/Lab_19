# Challenge Snake.apk — Writeup Reverse Engineering & Exploitation SnakeYAML

## 🎯 Objectif

L'application Android implémente plusieurs mécanismes de protection anti-reverse 
(détection de root, détection d'émulateur et détection de Frida via la librairie native). 
Elle lit un fichier YAML depuis le stockage externe et le parse en utilisant une version 
vulnérable de **SnakeYAML** (CVE-2022-1471).

L'objectif est d'exploiter cette vulnérabilité de désérialisation pour instancier une 
classe cachée nommée `BigBoss` qui génère et affiche le flag dans logcat.

- **Niveau :** Hard
- **Techniques :** Analyse statique (JADX + apktool), exploitation SnakeYAML (CVE-2022-1471), Intent ADB, logcat

---

## Étape 1 : Préparation de l'environnement

On commence par installer tous les outils nécessaires et préparer l'émulateur Android.
On utilise un émulateur **Pixel 6 API 37** (Google Play Store) — sur cette version,
SELinux fonctionne en mode permissif ce qui permet à l'app de se lancer sans déclencher 
les protections anti-émulateur.

### Récupérer l'APK et installer sur l'émulateur

```powershell
adb install snake.apk
```

<img width="414" height="475" alt="image" src="https://github.com/user-attachments/assets/1a47a40c-b1bb-4712-9c2b-4638929328b4" />

<img width="229" height="437" alt="image" src="https://github.com/user-attachments/assets/8c8f444e-3e64-48fe-936c-608c0bb4d82b" />

### Vérification de l'environnement

Avant de commencer, on vérifie que tous les outils sont bien installés.

#### JADX-GUI — pour décompiler l'APK en Java lisible

<img width="628" height="65" alt="image" src="https://github.com/user-attachments/assets/44f524f0-d1ec-49a6-845f-5d174ec31c70" />

<img width="927" height="515" alt="image" src="https://github.com/user-attachments/assets/a89035e7-fb41-4e74-8e74-21235b1fbd54" />

#### Java — requis pour exécuter apktool et uber-apk-signer

<img width="599" height="65" alt="image" src="https://github.com/user-attachments/assets/7cb78338-5572-4207-a04a-8e2e859959a8" />

#### Apktool — pour décompiler/recompiler l'APK en Smali

<img width="623" height="41" alt="image" src="https://github.com/user-attachments/assets/e298eb0f-fc1d-46cc-9b87-0261185ecb56" />

#### ADB — pour communiquer avec l'émulateur

<img width="706" height="110" alt="image" src="https://github.com/user-attachments/assets/70c72b87-fb69-4bce-acb7-a256e3481bb7" />

> ✅ Tous les outils sont vérifiés et l'environnement est prêt. L'APK est installé sur 
> l'émulateur **Pixel 6 API 37** — l'application se lance et affiche l'écran Snake 
> sans déclencher les protections anti-émulateur sur cette version d'Android.

---

## Étape 2 : Analyse statique approfondie avec JADX-GUI

On ouvre `snake.apk` dans JADX-GUI et on explore le code source décompilé pour 
comprendre le flux de l'application et identifier les classes importantes.

### Classe d'entrée : MainActivity

<img width="959" height="485" alt="image" src="https://github.com/user-attachments/assets/bc90175a-f9bb-4ae2-a3ec-abef3839551e" />

La logique principale dans `onCreate` :
- Vérifie la présence d'un **Intent extra** nommé `SNAKE` avec la valeur exacte `BigBoss`
- Si la condition est remplie, lit le fichier `/sdcard/Snake/Skull_Face.yml`
- Le contenu est parsé avec **SnakeYAML** (classe `Yaml`)
- Plusieurs méthodes de détection sont présentes : `checkForDangerousBinaries()`, 
  `checkForRootManagementApps()`, `checkForRootShell()`, `isDeviceRooted()`

### Classe importante : BigBoss

<img width="959" height="481" alt="image" src="https://github.com/user-attachments/assets/109850d2-26ac-4dd4-8cfa-2f8d81245a66" />

C'est la cible de notre exploitation :
- Charge la librairie native via `System.loadLibrary("snake")`
- Si le paramètre reçu est exactement `Snaaaaaaaaaaaaaake`, appelle `stringFromJNI()`
- La fonction JNI génère le flag en hexadécimal, le convertit en ASCII et l'affiche 
  via `Log.d("BigBoss: ", flag)` dans logcat

### Protections détectées

| Protection | Mécanisme |
|---|---|
| **Root** | Vérification `Build.TAGS`, `/system/app/Superuser.apk`, commande `su` |
| **Émulateur** | Propriétés `ro.hardware`, `ro.product.model` |
| **Frida** | Scan de processus et ports dans `libsnake.so` |
| **Anti-debug** | Appel `ptrace` dans la librairie native |

> 💡 Le développeur a caché la logique du flag dans `BigBoss`, une classe jamais 
> appelée normalement. La vulnérabilité **SnakeYAML (CVE-2022-1471)** permet de 
> forcer son instanciation avec le bon paramètre via un fichier YAML malveillant.

---

## Étape 3 : Patch Smali (non nécessaire dans notre cas)

> ℹ️ Cette étape de patching Smali **n'est pas nécessaire** sur **Pixel 6 API 37** —
> l'émulateur n'est pas rooté et l'application se lance normalement sans déclencher 
> les protections anti-root/émulateur. SELinux fonctionne en mode permissif pour ptrace,
> ce qui permet à la librairie native de s'exécuter sans crasher.

Si tu utilises un émulateur rooté ou une version Android plus ancienne, il faudrait :
1. Décompiler avec `apktool d snake.apk -o snake_smali`
2. Patcher `MainActivity.smali` et `BigBoss.smali` pour neutraliser le `loadLibrary`
3. Recompiler avec `apktool b snake_smali -o snake_patched.apk`
4. Signer avec `uber-apk-signer` et réinstaller

---

## Étape 4 : Création du payload YAML (exploitation CVE-2022-1471)

La vulnérabilité **SnakeYAML CVE-2022-1471** permet une désérialisation arbitraire —
on peut forcer l'instanciation de n'importe quelle classe Java via un fichier YAML.

### Payload utilisé

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

- `!!com.pwnsec.snake.BigBoss` → force SnakeYAML à instancier la classe `BigBoss`
- `["Snaaaaaaaaaaaaaake"]` → paramètre passé au constructeur, déclenchant l'appel JNI

### 1. Créer le dossier sur l'émulateur

```powershell
adb shell mkdir -p /sdcard/Snake
```

<img width="644" height="35" alt="image" src="https://github.com/user-attachments/assets/e58addf0-7b44-4e96-8650-da9825cea2d9" />

### 2. Créer et transférer le fichier YAML

```powershell
# Créer le fichier en local
Set-Content -Path "C:\Users\HP\Downloads\Skull_Face.yml" `
  -Value '!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]'

# Transférer sur l'émulateur
adb push C:\Users\HP\Downloads\Skull_Face.yml /sdcard/Snake/Skull_Face.yml

# Vérifier le contenu
adb shell cat /sdcard/Snake/Skull_Face.yml
```

<img width="959" height="188" alt="image" src="https://github.com/user-attachments/assets/39e8a5b2-ad7e-4158-802e-196dfe0b4442" />

### 3. Accorder les permissions de stockage

```powershell
adb shell pm grant com.pwnsec.snake android.permission.READ_EXTERNAL_STORAGE
```

<img width="847" height="211" alt="image" src="https://github.com/user-attachments/assets/19e285e7-e472-42f4-ab23-f81867952095" />

---

## Étape 5 : Lancement de l'application avec l'Intent approprié

On démarre `MainActivity` en passant l'extra Intent `SNAKE=BigBoss` qui déclenche 
la lecture du fichier YAML et la désérialisation de `BigBoss`.

```powershell
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss --activity-clear-task
```

<img width="887" height="59" alt="image" src="https://github.com/user-attachments/assets/fc9ac089-f184-4e47-9978-472d61e8060d" />

---

## Étape 6 : Récupération du flag via logcat

Le flag n'est pas affiché à l'écran — il est imprimé dans les logs système via `Log.d()`.

```powershell
adb logcat | Select-String -Pattern "BigBoss|PWNSEC"
```

<img width="872" height="361" alt="image" src="https://github.com/user-attachments/assets/e960e124-85d0-4956-aa1b-7dd21c3178a4" />

---

## 🏁 Flag

 








