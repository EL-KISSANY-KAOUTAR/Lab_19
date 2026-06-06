# Challenge Snake.apk — Writeup Reverse Engineering & Exploitation SnakeYAML

## 🎯 Objectif

L'application Android implémente plusieurs mécanismes de protection anti-reverse (détection de root, détection d'émulateur et détection de Frida via la librairie native). Elle lit un fichier YAML depuis le stockage externe et le parse en utilisant une version vulnérable de **SnakeYAML** (CVE-2022-1471).

L'objectif est d'exploiter cette vulnérabilité de désérialisation pour instancier une classe cachée nommée `BigBoss` qui génère et affiche le flag dans logcat.

## Étape 1 : Préparation de l'environnement

### Récupérer l'APK et installer sur l'émulateur

<img width="367" height="386" alt="image" src="https://github.com/user-attachments/assets/7a9c7161-55f2-4ebf-8375-c7a452567aaa" />


### Vérification de l'environnement

#### JADX-GUI

<img width="628" height="65" alt="image" src="https://github.com/user-attachments/assets/44f524f0-d1ec-49a6-845f-5d174ec31c70" />

<img width="927" height="515" alt="image" src="https://github.com/user-attachments/assets/a89035e7-fb41-4e74-8e74-21235b1fbd54" />

Java

<img width="599" height="65" alt="image" src="https://github.com/user-attachments/assets/7cb78338-5572-4207-a04a-8e2e859959a8" />

Apktool

<img width="623" height="41" alt="image" src="https://github.com/user-attachments/assets/e298eb0f-fc1d-46cc-9b87-0261185ecb56" />

Adb

<img width="706" height="110" alt="image" src="https://github.com/user-attachments/assets/70c72b87-fb69-4bce-acb7-a256e3481bb7" />

Installe l’APK original pour tester les comportements :

<img width="414" height="475" alt="image" src="https://github.com/user-attachments/assets/1a47a40c-b1bb-4712-9c2b-4638929328b4" />

<img width="229" height="437" alt="image" src="https://github.com/user-attachments/assets/8c8f444e-3e64-48fe-936c-608c0bb4d82b" />

Analyse statique approfondie avec Jadx

Classe d’entrée : MainActivity

<img width="959" height="485" alt="image" src="https://github.com/user-attachments/assets/bc90175a-f9bb-4ae2-a3ec-abef3839551e" />

Classe importante : BigBoss (dans com.pwnsec.snake.BigBoss)

<img width="959" height="481" alt="image" src="https://github.com/user-attachments/assets/109850d2-26ac-4dd4-8cfa-2f8d81245a66" />

Note : Cette étape de patching Smali n'est pas nécessaire dans notre cas, car l'émulateur utilisé (Pixel 6 API 37) n'est pas rooté et l'application se lance normalement sans déclencher les protections anti-root/émulateur.

Étape 4 : Création du payload YAML
1. Créer le dossier sur l'émulateur

<img width="644" height="35" alt="image" src="https://github.com/user-attachments/assets/e58addf0-7b44-4e96-8650-da9825cea2d9" />

crée et transfère le fichier YAML :

<img width="959" height="188" alt="image" src="https://github.com/user-attachments/assets/39e8a5b2-ad7e-4158-802e-196dfe0b4442" />


<img width="847" height="211" alt="image" src="https://github.com/user-attachments/assets/19e285e7-e472-42f4-ab23-f81867952095" />

Lancement avec l'Intent

<img width="887" height="59" alt="image" src="https://github.com/user-attachments/assets/fc9ac089-f184-4e47-9978-472d61e8060d" />

Récupération du flag via logcat

<img width="872" height="361" alt="image" src="https://github.com/user-attachments/assets/e960e124-85d0-4956-aa1b-7dd21c3178a4" />

 








