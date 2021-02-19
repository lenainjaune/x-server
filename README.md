# x-server
HOWTO pour serveur X

# X cascadé sur autre session (Debian only)
Contexte : quand un utilisateur a des droits sur l'affichage graphique de sa session, il n'a généralement pas les droits sur une autre session. Par exemple, sur une simple installation de Debian Buster fraichement installée avec un gestionnaire de bureau, si on exécute ```xeyes``` depuis un terminal depuis sa session graphique, l'application se lance alors mais si on passe en root la même commande abouti au bien connu ```Error: Can't open display: ...```. De même quand on accède à une session SSH par X-forwarding (```ssh -X ou -Y```) on ne peut lancer des applications graphique que depuis la session de l'utilisateur avec lequel on s'est loggué.

Ici est expliqué une méthode de contournement personnelle (à priori sans risque) pour "cascader" les droits d'une session utilisateur à une autre (root par exemple). Cette méthode n'est pas certifiée fiable, ni sûrement la plus appropriée, mais pour moi elle a fonctionné.

Comment ça marche (de ce que j'ai compris) : pour autoriser le serveur X dans une session graphique, il faut que la variable d'environnement DISPLAY de la session en cours corresponde à un des "magic cookie" d'autorisations d'affichage X (```xauth```). Chaque cookie référence un affichage unique (session locale et SSH par exemple).

Commande tout-en-un:
```sh
user@host:~$ C=$( xauth list |grep $( echo $DISPLAY |grep -Eo :[0-9]+ ) ) \
su -w C,DISPLAY -c "xauth add \$C ; app args" -
```
Explications :
``` C=$( ... )``` sauvegarde le cookie de la session actuelle pour l'affichage actuel, ```echo $DISPLAY |grep -Eo :[0-9]+ )``` identifie l'affichage de la session actuelle sous la forme *:c...c* où chaque *c* est un chiffre qui composent un nombre et ce qui suit est ignoré (filtre avec une regex (```-E```), uniquement (```-o```) ce qui commence par ":" suivi de 1 à plusieurs chiffres (```:[0-9]+```)), ```xauth list | grep ...``` extrait le cookie de la session actuelle, ```xauth add \$C``` ajoute le cookie s'il n'existe pas (on doit échapper \$C pour que son expansion ait lieue pendant l'exécution de la commande en root (```su .... -```) et non avant, ```su -w C,DISPLAY -c '...' -``` permet de passer en root avec son environnement (```-``` final) une commande (```-c```) tout en conservant les variables DISPLAY et le cookie C (```-w``` = --whitelist-environment) enfin ```app args``` est l'application à exécuter sans, ou avec 1 ou plusieurs arguments séparés par des espaces.

Exemples :

```C=$( xauth list |grep $( echo $DISPLAY |grep -Eo :[0-9]+ ) ) su -w C,DISPLAY -c "xauth add \$C ; xeyes" -```  
exécute ```xeyes``` (normalement installé avec le serveur X)

```C=$( xauth list |grep $( echo $DISPLAY |grep -Eo :[0-9]+ ) ) su -w C,DISPLAY -c "xauth add \$C ; thunar /mnt" -```   ouvre le gestionnaire de fichier ```thunar``` depuis le dossier ```/mnt```

## Ancienne méthode (Debian only)
Commande :
```sh
U=$USER su -w DISPLAY,U -c 'cascade_x_app.sh app args' -
```
Explications : ```U=$USER ...``` permet de sauvegarder le user actuel dans la commande actuelle, ```su -w DISPLAY,U -c '...' -``` permet de passer en root avec son environnement (```-``` final) une commande (```-c```) tout en conservant les variables DISPLAY et U (```-w``` = --whitelist-environment), ```cascade_x_app.sh app args``` exécute le script (voir dessous) en lui fournissant l'application à exécuter et ses arguments

=> on devra saisir le mot de passe de root

Exemples :

```U=$USER su -w DISPLAY,U -c 'cascade_x_app.sh xeyes' -``` exécute ```xeyes``` (normalement installé avec le serveur X)

```U=$USER su -w DISPLAY,U -c 'cascade_x_app.sh thunar /mnt' -``` ouvre le gestionnaire de fichier ```thunar``` depuis le dossier ```/mnt```

Le script :
```sh
user@host:~$ cat cascade_x_app.sh
#! /bin/bash
#echo $U $DISPLAY $USER ; xauth list
o_id_display=$( echo $DISPLAY |grep -Eo :[0-9]+ )
o_cookie=$( su -c "xauth list |grep $o_id_display" - $U )
xauth add $o_cookie
exec "$@"
```
Explications : ```o_id_display``` identifie l'affichage de la session d'origine (filtre avec une regex (```-E```), uniquement (```-o```) ce qui commence par ":" suivi de 1 à plusieurs chiffres (```:[0-9]+```)), ```o_cookie``` identifie le cookie de la session d'origine (se connecte temporairement à la session d'origine par le user ```$U``` (```su ... - $U```) en récupérant le cookie des autorisations d'affichage X (```xauth list```) de son affichage d'origine ; depuis root pas de demande de mot de passe), ```xauth add``` ajoute le cookie, ```exec "$@"``` exécute la commande fournie en ```$@``` c'est à dire tous les arguments ```$1``` à ```$n```
