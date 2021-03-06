# Teaching-HEIGVD-SRX-2020-Laboratoire-IDS

Authors : Dupont Maxime, Florian Mulhauser

**Ce travail de laboratoire est à faire en équipes de 2 personnes** (oui... en remote...). Je vous laisse vous débrouiller ;-)

**ATTENTION : Commencez par créer un Fork de ce repo et travaillez sur votre fork.**

Clonez le repo sur votre machine. Vous pouvez répondre aux questions en modifiant directement votre clone du README.md ou avec un fichier pdf que vous pourrez uploader sur votre fork.

**Le rendu consiste simplement à répondre à toutes les questions clairement identifiées dans le text avec la mention "Question" et à les accompagner avec des captures. Le rendu doit se faire par une "pull request". Envoyer également le hash du dernier commit et votre username GitHub par email au professeur et à l'assistant**

## Table de matières

[Introduction](#introduction)

[Echéance](#echéance)

[Configuration du réseau](#configuration-du-réseau-sur-virtualbox)

[Installation de Snort](#installation-de-snort-sur-linux)

[Essayer Snort](#essayer-snort)

[Utilisation comme IDS](#utilisation-comme-un-ids)

[Ecriture de règles](#ecriture-de-règles)

[Travail à effectuer](#exercises)


## Echéance

Ce travail devra être rendu le dimanche après la fin de la 2ème séance de laboratoire, soit au plus tard, **le 6 avril 2020, à 23h59.**


## Introduction

Dans ce travail de laboratoire, vous allez explorer un système de detection contre les intrusions (IDS) dont l'utilisation es très répandue grace au fait qu'il est gratuit et open source. Il s'appelle [Snort](https://www.snort.org). Il existe des versions de Snort pour Linux et pour Windows.

### Les systèmes de detection d'intrusion

Un IDS peut "écouter" tout le traffic de la partie du réseau où il est installé. Sur la base d'une liste de règles, il déclenche des actions sur des paquets qui correspondent à la description de la règle.

Un exemple de règle pourrait être, en language commun : "donner une alerte pour tous les paquets envoyés par le port http à un serveur web dans le réseau, qui contiennent le string 'cmd.exe'". En on peut trouver des règles très similaires dans les règles par défaut de Snort. Elles permettent de détecter, par exemple, si un attaquant essaie d'executer un shell de commandes sur un serveur Web tournant sur Windows. On verra plus tard à quoi ressemblent ces règles.

Snort est un IDS très puissant. Il est gratuit pour l'utilisation personnelle et en entreprise, où il est très utilisé aussi pour la simple raison qu'il est l'un des plus efficaces systèmes IDS.

Snort peut être exécuté comme un logiciel indépendant sur une machine ou comme un service qui tourne après chaque démarrage. Si vous voulez qu'il protège votre réseau, fonctionnant comme un IPS, il faudra l'installer "in-line" avec votre connexion Internet.

Par exemple, pour une petite entreprise avec un accès Internet avec un modem simple et un switch interconnectant une dizaine d'ordinateurs de bureau, il faudra utiliser une nouvelle machine executant Snort et placée entre le modem et le switch.


## Matériel

Vous avez besoin de votre ordinateur avec VirtualBox et une VM Kali Linux. Vous trouverez un fichier OVA pour la dernière version de Kali sur `//eistore1/cours/iict/Laboratoires/SRX/Kali` si vous en avez besoin.


## Configuration du réseau sur VirtualBox

Votre VM fonctionnera comme IDS pour "protéger" votre machine hôte (par exemple, si vous faites tourner VirtualBox sur une machine Windows, Snort sera utilisé pour capturer le trafic de Windows vers l'Internet).

Pour cela, il faudra configurer une réseau de la VM en mode "bridge" et activer l'option "Promiscuous Mode" dans les paramètres avancés de l'interface. Le mode bridge dans l'école ne vous permet pas d'accéder à l'Internet depuis votre VM. Vous pouvez donc rajouter une deuxième interface réseau à votre Kali configurée comme NAT. La connexion Internet est indispensable pour installer Snort mais pas vraiment nécessaire pour les manipulations du travail pratique.

Pour les captures avec Snort, assurez-vous de toujours indiquer la bonne interface dans la ligne de commandes, donc, l'interface configurée en mode promiscuous.

![Topologie du réseau virtualisé](images/Snort_Kali.png)


## Installation de Snort sur Linux

On va installer Snort sur Kali Linux. Si vous avez déjà une VM Kali, vous pouvez l'utiliser. Sinon, vous avez la possibilité de copier celle sur `eistore`.

La manière la plus simple c'est de d'installer Snort en ligne de commandes. Il suffit d'utiliser la commande suivante :

```
sudo apt update && apt install snort
```

Ceci télécharge et installe la version la plus récente de Snort.

Vers la fin de l'installation, on vous demande de fournir l'adresse de votre réseau HOME. Il s'agit du réseau que vous voulez protéger. Cela sert à configurer certaines variables pour Snort. Pour les manipulations de ce laboratoire, vous pouvez donner n'importe quelle adresse comme réponse.


## Essayer Snort

Une fois installé, vous pouvez lancer Snort comme un simple "sniffer". Pourtant, ceci capture tous les paquets, ce qui peut produire des fichiers de capture énormes si vous demandez de les journaliser. Il est beaucoup plus efficace d'utiliser des règles pour définir quel type de trafic est intéressant et laisser Snort ignorer le reste.

Snort se comporte de différentes manières en fonction des options que vous passez en ligne de commande au démarrage. Vous pouvez voir la grande liste d'options avec la commande suivante :

```
snort --help
```

On va commencer par observer tout simplement les entêtes des paquets IP utilisant la commande :

```
snort -v -i eth0
```

**ATTENTION : assurez-vous de bien choisir l'interface qui se trouve en mode bridge/promiscuous. Elle n'est peut-être pas eth0 chez-vous!**

Snort s'execute donc et montre sur l'écran tous les entêtes des paquets IP qui traversent l'interface eth0. Cette interface est connectée à l'interface réseau de votre machine hôte à travers le bridge de VirtualBox.

Pour arrêter Snort, il suffit d'utiliser `CTRL-C` (**attention** : il peut arriver de temps à autres que snort ne réponde pas correctement au signal d'arrêt. Dans ce cas-là, il faudra utiliser `kill` pour arrêter le process).

## Utilisation comme un IDS

Pour enregistrer seulement les alertes et pas tout le trafic, on execute Snort en mode IDS. Il faudra donc spécifier un fichier contenant des règles.

Il faut noter que `/etc/snort/snort.config` contient déjà des références aux fichiers de règles disponibles avec l'installation par défaut. Si on veut tester Snort avec des règles simples, on peut créer un fichier de config personnalisé (par exemple `mysnort.conf`) et importer un seul fichier de règles utilisant la directive "include".

Les fichiers de règles sont normalement stockes dans le repertoire `/etc/snort/rules/`, mais en fait un fichier de config et les fichiers de règles peuvent se trouver dans n'importe quel repertoire.

Par exemple, créez un fichier de config `mysnort.conf` dans le repertoire `/etc/snort` avec le contenu suivant :

```
include /etc/snort/rules/icmp2.rules
```

Ensuite, créez le fichier de règles `icmp2.rules` dans le repertoire `/etc/snort/rules/` et rajoutez dans ce fichier le contenu suivant :

`alert icmp any any -> any any (msg:"ICMP Packet"; sid:4000001; rev:3;)`

On peut maintenant executer la commande :

```
snort -c /etc/snort/mysnort.conf
```

Vous pouvez maintenant faire quelques pings depuis votre hôte et regarder les résultas dans le fichier d'alertes contenu dans le repertoire `/var/log/snort/`.


## Ecriture de règles

Snort permet l'écriture de règles qui décrivent des tentatives de exploitation de vulnérabilités bien connues. Les règles Snort prennent en charge à la fois, l'analyse de protocoles et la recherche et identification de contenu.

Il y a deux principes de base à respecter :

* Une règle doit être entièrement contenue dans une seule ligne
* Les règles sont divisées en deux sections logiques : (1) l'entête et (2) les options.

L'entête de la règle contient l'action de la règle, le protocole, les adresses source et destination, et les ports source et destination.

L'option contient des messages d'alerte et de l'information concernant les parties du paquet dont le contenu doit être analysé. Par exemple:

```
alert tcp any any -> 192.168.1.0/24 111 (content:"|00 01 86 a5|"; msg: "mountd access";)
```

Cette règle décrit une alerte générée quand Snort trouve un paquet avec tous les attributs suivants :

* C'est un paquet TCP
* Emis depuis n'importe quelle adresse et depuis n'importe quel port
* A destination du réseau identifié par l'adresse 192.168.1.0/24 sur le port 111

Le text jusqu'au premier parenthèse est l'entête de la règle.

```
alert tcp any any -> 192.168.1.0/24 111
```

Les parties entre parenthèses sont les options de la règle:

```
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

Les options peuvent apparaître une ou plusieurs fois. Par exemple :

```
alert tcp any any -> any 21 (content:"site exec"; content:"%"; msg:"site
exec buffer overflow attempt";)
```

La clé "content" apparait deux fois parce que les deux strings qui doivent être détectés n'apparaissent pas concaténés dans le paquet mais à des endroits différents. Pour que la règle soit déclenchée, il faut que le paquet contienne **les deux strings** "site exec" et "%".

Les éléments dans les options d'une règle sont traitées comme un AND logique. La liste complète de règles sont traitées comme une succession de OR.

## Informations de base pour le règles

### Actions :

```
alert tcp any any -> any any (msg:"My Name!"; content:"Skon"; sid:1000001; rev:1;)
```

L'entête contient l'information qui décrit le "qui", le "où" et le "quoi" du paquet. Ça décrit aussi ce qui doit arriver quand un paquet correspond à tous les contenus dans la règle.

Le premier champ dans le règle c'est l'action. L'action dit à Snort ce qui doit être fait quand il trouve un paquet qui correspond à la règle. Il y a six actions :

* alert - générer une alerte et écrire le paquet dans le journal
* log - écrire le paquet dans le journal
* pass - ignorer le paquet
* drop - bloquer le paquet et l'ajouter au journal
* reject - bloquer le paquet, l'ajouter au journal et envoyer un `TCP reset` si le protocole est TCP ou un `ICMP port unreachable` si le protocole est UDP
* sdrop - bloquer le paquet sans écriture dans le journal

### Protocoles :

Le champ suivant c'est le protocole. Il y a trois protocoles IP qui peuvent être analysez par Snort : TCP, UDP et ICMP.


### Adresses IP :

La section suivante traite les adresses IP et les numéros de port. Le mot `any` peut être utilisé pour définir "n'import quelle adresse". On peut utiliser l'adresse d'une seule machine ou un block avec la notation CIDR.

Un opérateur de négation peut être appliqué aux adresses IP. Cet opérateur indique à Snort d'identifier toutes les adresses IP sauf celle indiquée. L'opérateur de négation est le `!`.

Par exemple, la règle du premier exemple peut être modifiée pour alerter pour le trafic dont l'origine est à l'extérieur du réseau :

```
alert tcp !192.168.1.0/24 any -> 192.168.1.0/24 111
(content: "|00 01 86 a5|"; msg: "external mountd access";)
```

### Numéros de Port :

Les ports peuvent être spécifiés de différentes manières, y-compris `any`, une définition numérique unique, une plage de ports ou une négation.

Les plages de ports utilisent l'opérateur `:`, qui peut être utilisé de différentes manières aussi :

```
log udp any any -> 192.168.1.0/24 1:1024
```

Journaliser le traffic UDP venant d'un port compris entre 1 et 1024.

--

```
log tcp any any -> 192.168.1.0/24 :6000
```

Journaliser le traffic TCP venant d'un port plus bas ou égal à 6000.

--

```
log tcp any :1024 -> 192.168.1.0/24 500:
```

Journaliser le traffic TCP venant d'un port privilégié (bien connu) plus grand ou égal à 500 mais jusqu'au port 1024.


### Opérateur de direction

L'opérateur de direction `->`indique l'orientation ou la "direction" du trafique.

Il y a aussi un opérateur bidirectionnel, indiqué avec le symbole `<>`, utile pour analyser les deux côtés de la conversation. Par exemple un échange telnet :

```
log 192.168.1.0/24 any <> 192.168.1.0/24 23
```

## Alertes et logs Snort

Si Snort détecte un paquet qui correspond à une règle, il envoie un message d'alerte ou il journalise le message. Les alertes peuvent être envoyées au syslog, journalisées dans un fichier text d'alertes ou affichées directement à l'écran.

Le système envoie **les alertes vers le syslog** et il peut en option envoyer **les paquets "offensifs" vers une structure de repertoires**.

Les alertes sont journalisées via syslog dans le fichier `/var/log/snort/alerts`. Toute alerte se trouvant dans ce fichier aura son paquet correspondant dans le même repertoire, mais sous le fichier `snort.log.xxxxxxxxxx` où `xxxxxxxxxx` est l'heure Unix du commencement du journal.

Avec la règle suivante :

```
alert tcp any any -> 192.168.1.0/24 111
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

un message d'alerte est envoyé à syslog avec l'information "mountd access". Ce message est enregistré dans `/var/log/snort/alerts` et le vrai paquet responsable de l'alerte se trouvera dans un fichier dont le nom sera `/var/log/snort/snort.log.xxxxxxxxxx`.

Les fichiers log sont des fichiers binaires enregistrés en format pcap. Vous pouvez les ouvrir avec Wireshark ou les diriger directement sur la console avec la commande suivante :

```
tcpdump -r /var/log/snort/snort.log.xxxxxxxxxx
```

Vous pouvez aussi utiliser des captures Wireshark ou des fichiers snort.log.xxxxxxxxx comme source d'analyse por Snort.

## Exercises

**Réaliser des captures d'écran des exercices suivants et les ajouter à vos réponses.**

> Nous avons rajouté une brève annexe à la fin de ce document veuillez bien y preter attention s'il vous plait.

### Essayer de répondre à ces questions en quelques mots :

**Question 1: Qu'est ce que signifie les "preprocesseurs" dans le contexte de Snort ?**

---

**Reponse :**  

> Les preprocesseurs ont pour but de préparer les paquets qu'ils examinent pour pouvoir les envoyer au moteur de détection. Il y a deux utilités: analyser pour detecter des activités malicieuses, et l'autre est de modifier les paquets pour qu'ils soient bien interprétés par le moteur de détection.

---

**Question 2: Pourquoi êtes vous confronté au WARNING suivant `"No preprocessors configured for policy 0"` lorsque vous exécutez la commande `snort` avec un fichier de règles ou de configuration "home-made" ?**

---

**Reponse :**  

> Cela vient du fait que snort ne charge pas par défaut le préprocesseur situé dans /etc/snort/snort.conf. De plus nous n'avons pas configuré nous-même un préprocesseur.
---

--

### Trouver votre nom :

Considérer la règle simple suivante:

alert tcp any any -> any any (msg:"Mon nom!"; content:"Rubinstein"; sid:4000015; rev:1;)

**Question 3: Qu'est-ce qu'elle fait la règle et comment ça fonctionne ?**

---

**Reponse :**  

> Nous allons lister de gauche à droite:
* `alert` : cette règle, si remplie va retourner une alerte et écrire dans un log.
* `tcp any any -> any any` : Ça s'applique à tous les paquets TCP sur le réseau (de toutes les IPs et tous les ports)
* `msg: "Mon nom!"; content:"Rubinstein"` : Va écrire dans le journal "Mon nom !" si la chaine de charactère ASCII (en clair donc) "Rubinstein" est contenue et lisible dans un des paquets.

> Donc si un des paquets TCP passant sur le réseau (toutes IPs, tout ports) contient la chaine "Rubinstein", cela va créer une alerte nomée "Mon nom!" et l'écrit dans le journal.
---

Utiliser un éditeur et créer un fichier `myrules.rules` sur votre répertoire home. Rajouter une règle comme celle montrée avant mais avec votre nom ou un mot clé de votre préférence. Lancer snort avec la commande suivante :

```
sudo snort -c myrules.rules -i eth0
```

**Question 4: Que voyez-vous quand le logiciel est lancé ? Qu'est-ce que tous les messages affichés veulent dire ?**

---

**Reponse :**  

> On nous affiche le fichier de règles chargés, le directory où sera stocké les logs, les ports utilisés, les patterns matchings.


![](screenshots/Q4.1.PNG)

![](screenshots/Q4.2.PNG)



---

Aller à un site web contenant dans son text votre nom ou votre mot clé que vous avez choisi (il faudra chercher un peu pour trouver un site en http...).

**Question 5: Que voyez-vous sur votre terminal quand vous visitez le site ?**

---
> ![](screenshots/Q4.3.PNG)


**Reponse :**  
> Nous voyons plein de `WARNING: No preprocessor configured for policy 0`, c'est parce qu'on n'a pas de préprocesseurs. Cependant rien de supplémentaire, nous auront plus de détails à l'arrêt de snort.
---

Arrêter Snort avec `CTRL-C`.

**Question 6: Que voyez-vous quand vous arrêtez snort ? Décrivez en détail toutes les informations qu'il vous fournit.**

---

**Reponse :**  

> On a ensuite un résumé sur pleins de stats des paquets: leur nombre et taille en mémoire, leurs états I/O(reçus, filtrés, analysés, rejeté, etc), leurs protocoles. Enfin un résumé des stats de ce que la règle a fait (nombre d'alert, de log, des choses bloquées ou non etc).

> ![](screenshots/Q4.4.PNG)

> ![](screenshots/Q4.5.PNG)

---


Aller au répertoire /var/log/snort. Ouvrir le fichier `alert`. Vérifier qu'il y ait des alertes pour votre nom ou mot choisi.

**Question 7: A quoi ressemble l'alerte ? Qu'est-ce que chaque élément de l'alerte veut dire ? Décrivez-la en détail !**

---

**Reponse :**  

> L'alerte s'écrit enplusieurs lignes, comme suit:
* on a le SID de la règle et son nom (renseigné à `msg:"Myname"`)
* On a le niveau de priority de l'alerte, par défaut c'et à 0, comme ici.
* On a l'heure UNIX de l'alerte, donc date et heure précise jusqu'à la nanosecondes. On nous indique de quel IP:PORT vers quel IP:PORT ça a été envoyés
*  Ou nous donne ensuite le protocole du paquets ainsi que des renseignements sur les infos du paquets, les longueurs les sécurité. C'est un peu les headers du paquets.
* en dernier des infos sur les options de TCP
---


--

### Detecter une visite à Wikipedia

Ecrire une règle qui journalise (sans alerter) un message à chaque fois que Wikipedia est visité **DEPUIS VOTRE** station. **Ne pas utiliser une règle qui détecte un string ou du contenu**.

**Question 8: Quelle est votre règle ? Où le message a-t'il été journalisé ? Qu'est-ce qui a été journalisé ?**

---

**Reponse :**  

> la règle: `log tcp 192.168.1.42 any -> 91.198.174.192 443 (msg:"Wikipedia visited"; sid:4242424242; rev:2)`

> On obtient les IP de notre poste et de Wikipedia, en effectuant une commande ping vers wikipedia(`ping www.wikipedia.com`). Ensuite on met comme port de destination 443, pour logguer que les visites (TCP handshakes d'https). On met l'option log pour avoir juste les logs mais pas d'alertes.

> Les messages sont journalisés dan le dossier /var/log/snort dans un fichier crée snort.log.xxxxxxxxx avec l'heure Linux.
---

--

### Detecter un ping d'un autre système

Ecrire une règle qui alerte à chaque fois que votre système reçoit un ping depuis une autre machine (je sais que la situation actuelle du Covid-19 ne vous permet pas de vous mettre ensemble... utilisez votre imagination pour trouver la solution à cette question !). Assurez-vous que **ça n'alerte pas** quand c'est vous qui envoyez le ping vers un autre système !

**Question 9: Quelle est votre règle ?**

---

**Reponse :**  

> La règle est : `alert icmp !192.168.1.42 any -> 192.168.1.42 any (msg:"On me ping"; itype:8; sid: 0000000001; rev:1)`

> On utilise le !IP pour dire qu'on veut avoir en source tout sauf notre IP, qui par contre doit etre la destinataire. Le protocole des pings est ICMP. On indique `itype:8` pour ne pas recevoir des alertes quand une machine réponderais à un ping de notre machine.
---


**Question 10: Comment avez-vous fait pour que ça identifie seulement les pings entrants ?**

---

**Reponse :**  

> On a bien spécifié que ça devait venir de n'importe quel IP (autre que le notre), vers notre IP. De plus avec le itype:8, on précise qu'on ne veut pas etre alertés si une machine réponderais à un ping que notre machine a envoyé.
---


**Question 11: Où le message a-t-il été journalisé ?**

---

**Reponse :**  

> Comme indiqué au début du readme, le message est journalisé dans le fichier `/var/log/snort/alert`

---


**Question 12: Qu'est-ce qui a été journalisé ?**

---

**Reponse :**  

> On a entre autre dans alert: le sid de la règle, l'heure UNIX enregistrée de la réception du ping, les msg configuré, la priorité d'alerte, les IPs de source et dest .

> dans le snort.log.xxxxxxxxx on a l'heure, les IPS (source et dest), le type de requete donc icmp, et ensuite on a les .pcap qui contiennent les paquets qui ont déclenché la règle et donc l'alerte.
---

--

### Detecter les ping dans les deux sens

Modifier votre règle pour que les pings soient détectés dans les deux sens.

**Question 13: Qu'est-ce que vous avez modifié pour que la règle détecte maintenant le trafic dans les deux senses ?**

---

**Reponse :**  
> On change l'IP de source de !IPsrc à any et l'opérateur -> à <> pour faire dans les deux sens
`alert icmp any any <> 192.168.1.42 any (msg:"PING"; itype:8; sid: 0000000002; rev:1)`

> il ne faut pas enlever le itype:8, il permet de juste garder les envoie et pas toutes les réponses.
---


--

### Detecter une tentative de login SSH

Essayer d'écrire une règle qui Alerte qu'une tentative de session SSH a été faite depuis la machine d'un voisin (je sais que la situation actuelle du Covid-19 ne vous permet pas de vous mettre ensemble... utilisez votre imagination pour trouver la solution à cette question !). Si vous avez besoin de plus d'information sur ce qui décrit cette tentative (adresses, ports, protocoles), servez-vous de Wireshark pour analyser les échanges lors de la requête de connexion depuis votre voisi.

**Question 14: Quelle est votre règle ? Montrer la règle et expliquer en détail comment elle fonctionne.**

---

**Reponse :**  

> On veut detecter les SSH, on sait que ça utilise le port de destination 22, on va donc l'utiliser dans la règle.

`alert tcp any any -> 192.168.1.42 22 (msg:"SSH connexion reçue"; sid: 0000000003; rev:1)`

> On reçoit donc une alert lorsqu'un machine va tneter de se connecter à nous via SSH (donc IPdest c'est le notre et le port c'est 22 car SSH)
---


**Question 15: Montrer le message d'alerte enregistré dans le fichier d'alertes.**

---

**Reponse :**  
> n/a
---

--

### Analyse de logs

Lancer Wireshark et faire une capture du trafic sur l'interface connectée au bridge. Générez du trafic avec votre machine hôte qui corresponde à l'une des règles que vous avez ajouté à votre fichier de configuration personnel. Arrêtez la capture et enregistrez-la dans un fichier.

**Question 16: Quelle est l'option de Snort qui permet d'analyser un fichier pcap ou un fichier log ?**

---

**Reponse :**  

> `snort -r myFilePCAP.pcap` permet d'analyser un fichier pcap ou alors un fichier .log.xxxxxxxxx de la meme manière. Il faut donc l'option `-r`

---

Utiliser l'option correcte de Snort pour analyser le fichier de capture Wireshark.

**Question 17: Quelle est le comportement de Snort avec un fichier de capture ? Y-a-t'il une difference par rapport à l'analyse en temps réel ?**

---

**Reponse :**  

> Snort analyse le fichier de capture en prennant en compte ses règles, on a déjà tous les paquets donc on peut tout analyser d'un coup au lieu d'analyser tout le temps, les différents paquets qui arrivent.

---

**Question 18: Est-ce que des alertes sont aussi enregistrées dans le fichier d'alertes?**

---

**Reponse :**  

> Les alertes vont toujours être enregistrées dans son fichier, en effet snort agit comme s'il écoutait une interface.
---

--

### Contournement de la détection

Faire des recherches à propos des outils `fragroute` et `fragtest`.

**Question 20: A quoi servent ces deux outils ?**

---

**Reponse :**  
> Fragtest est contenu dans Fragroute, ces outils permettent de modifier le traffic pour tenter d'éviter un IDS.

> En lisant ce qui est dans https://tools.kali.org/information-gathering/fragroute. On peut voir que fragroute va pouvoir intercepter, modifier le traffic réseau qui va vers un hote. Cela se fait par exemple en mettant un delay, en duplicant ou en abandonnant des fragements. On peut aussi jouer sur des réordonnement, des overlaps etc.
---


**Question 21: Quel est le principe de fonctionnement ?**

---

**Reponse :**  
> Fragtest et Fragroute fragmentent, réodornnent ainsi que modifient le traffic afin d'essayer de contourner les règles de l'IDS
---


**Question 22: Qu'est-ce que le `Frag3 Preprocessor` ? A quoi ça sert et comment ça fonctionne ?**

---

**Reponse :**  
> Un preprocesseur de snort qui lui permet de contrecarrer la fragmentation des deux outils au-dessus, une manière de se "defendre" ou de palier a une faiblesse.
---


Reprendre l'exercice de la partie [Trouver votre nom](#trouver-votre-nom-). Essayer d'offusquer la détection avec `fragroute`.


**Question 23: Quel est le résultat de votre tentative ?**

---

**Reponse :**  
> n/a  ( Nous imaginons que l'evasion des règles est aisée puisque nous n'avons pris aucune mesure particulière pour bloquer ces outils).
---


Modifier le fichier `myrules.rules` pour que snort utiliser le `Frag3 Preprocessor` et refaire la tentative.


**Question 24: Quel est le résultat ?**

---

**Reponse :**  
> Nous avons eu des problèmes techniques lors de ce labo mais il nous semble probable que l'ajout de ce pré-processeur permet de bloquer l'évasion des règles (donc l'IDS fonctionne correctement).
---


**Question 25: A quoi sert le `SSL/TLS Preprocessor` ?**

---

**Reponse :**  
> Il sert à analyser les parties non-chiffrées de SSL/TLS.

---


**Question 26: A quoi sert le `Sensitive Data Preprocessor` ?**

---

**Reponse :**  
> Il permet de detecter et de filtrer la transmission des données sensibles comme les numéros de carte de crédit , les numéros de sécurité sociale ou encore les adresses e-mail.
---

### Conclusions


**Question 27: Donnez-nous vos conclusions et votre opinion à propos de snort**

---

**Reponse :**  
> Snort nous semble être un outil très puissant, cependant nous avons rencontré de nombreux problème que nous n'avons pas tous pu régler, il ne semble pas évident à maitriser.
---


### Annexe Important
Lors de la réalisation de ce labo nous avons rencontré de nombreux problèmes techniques, tout d'abord avec la VM, puis avec la configuration réseau et l'outil snort, ce qui a considérablement réduit notre motivation et notre capacité à avancer. Plutôt que de nous entêter sur des problèmes que nous n'arrivions pas à régler nous avons décider de nous aider des ressources en ligne du mieux possible pour donner des réponses théoriques et comprendre le fonctionnement et la logique de l'outil snort présenté. Nous n'avons cependant pu tester que très peu de manipulations en elle-mêmes à cause de ces problèmes, voila pourquoi il nous manque de nombreuses captures d'écrans dans ce README. En espérant que les labos suivant se déroulerons mieux pour nous. Merci de votre attention.


<sub>This guide draws heavily on http://cs.mvnu.edu/twiki/bin/view/Main/CisLab82014</sub>
