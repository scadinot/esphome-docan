# Docan — deux packs batterie DONOON 51,2 V

Configuration ESPHome interrogeant deux packs **DONOON 51,2 V** (BMS Daren
`YP02-16S200JC26`) en protocole **PYLON ASCII** sur RS485-A.

**La référence est le serveur Home Assistant**, dans `/config/esphome/`.
Ce dépôt est une copie versionnée. Les modifications partent du serveur.

---

## Matériel

| | |
|---|---|
| Carte | Waveshare ESP32-S3-ETH |
| Adresse IP | 192.168.0.71 |
| Pack 1 | adresse PYLON 01, **maître**, DIP tous OFF, firmware BMS V1.0.4 depuis le 29/09/2026 (V1.0.1 avant) |
| Pack 2 | adresse PYLON 02, esclave, DIP1 ON, firmware BMS V1.0.4 |
| Liaison | RS485-A, 9600 8N1, TX GPIO17 / RX GPIO16 |
| Capacités | 325,87 Ah et 325,00 Ah — 650,87 Ah au total |

**Ne pas toucher au RS485-B** : c'est le lien de parallélisme entre les deux
BMS.

## Organisation

```
dc-pack1      scripts et mesures du pack 1 (adresse 01)
dc-pack2      idem pack 2 (adresse 02)
dc-cellules   les 32 tensions de cellules, 16 par pack
dc-systeme    l'agrégat des deux packs
```

Une entité `PackN` se trouve dans `dc-packN`, sans exception : les quatre
`Energie totale/restante PackN` vivaient dans `dc-systeme` et ont été
rapatriées le 12/09/2026. `dc-systeme` ne garde que les versions `Systeme`.
Les noms d'entités n'ont pas changé, l'historique est continu.

La tension nominale de conversion Ah → kWh est la substitution
`tension_nominale` de `docan.yaml`, puisqu'elle sert maintenant dans trois
paquets.

Le protocole PYLON n'expose **aucune trame d'ensemble** — il interroge les
modules un par un. L'agrégat de `dc-systeme` est donc calculé côté ESP :

```
SOC Systeme = (restant₁ + restant₂) / (total₁ + total₂) × 100
```

Pondéré par la capacité, pas une moyenne des deux pourcentages. Tous les
calculs sont gardés par `isnan` : si un pack cesse de répondre, rien n'est
publié plutôt qu'une valeur calculée sur la moitié de la batterie.

## Un pack muet ne garde plus ses valeurs

Corrigé le 11/09/2026. Les capteurs des packs sont des `template` en
`update_interval: never` : ils valent NaN au démarrage, puis **ne le
redeviennent jamais d'eux-mêmes**. Un pack qui cessait de répondre gardait sa
dernière valeur indéfiniment, et le garde-fou `isnan` ci-dessus ne protégeait
donc que le premier cycle suivant un redémarrage.

Le script analogique sert maintenant de battement de cœur. Au **troisième
cycle sans réponse** — 90 s — tous les capteurs du pack repassent à NaN et les
libellés à « Pas de réponse ». Deux entités de diagnostic le signalent :

```
Communication Pack1   connectivity
Communication Pack2   connectivity
```

Trois et pas un : une trame perdue arrive, trois d'affilée non. Les seuils de
protection (CID2=47) ne sont pas invalidés — ce sont des constantes de
configuration, pas des mesures.

## La répartition ne ment plus quand les courants s'opposent

Corrigé le 11/09/2026. Le calcul était `fabs(i1) / (fabs(i1) + fabs(i2))`.
Exact quand les deux packs vont dans le même sens — le calcul algébrique qui
le remplace rend les mêmes chiffres, l'historique reste comparable.

Mais à `+1,0 A` et `−1,0 A`, les valeurs absolues affichaient **50 %**, le
partage parfait, alors qu'il ne passe rien vers l'onduleur et qu'un pack se
vide dans l'autre : le pire symptôme portait le meilleur chiffre. Et c'est
précisément le régime attendu à consommation nulle.

Désormais le dénominateur est `i1 + i2` — le vrai courant système — et rien
n'est publié si les packs vont en sens contraire au-delà de 0,3 A. Une entité
`Regime packs Systeme` nomme alors ce qui se passe :

```
Repos | Charge | Decharge
Transfert Pack1 vers Pack2 | Transfert Pack2 vers Pack1
Initialisation | Pas de reponse
```

Sans elle, une case vide serait indiscernable d'un pack muet. Les deux
derniers états se distinguent par les compteurs d'échec : au-delà de trois
cycles c'est un vrai silence, en-deçà c'est le démarrage.

## Lire la répartition : la pente, pas le pourcentage

L'ancienne grille de lecture disait qu'un relevé à fort courant tranche
— « vers 85 % à 40 A = résistance de contact, reprendre le câblage ».
Retirée le 12/09/2026 : les mesures montrent que le partage **oscille sur
plusieurs jours avec des inversions de sens**, ce qu'une asymétrie de
résistance, fixe par nature, ne peut pas produire.

Deux branches sur la même barre, convention positif = charge :

```
I1 - I2 = I_total x (R2 - R1)/(R1 + R2) - 2 x (OCV1 - OCV2)/(R1 + R2)
          \_________ pente __________/   \____ terme constant ____/
```

| Terme | Ce qu'il vaut | Ce qu'il signifie |
|---|---|---|
| Constant | écart de tension à vide | décalage en ampères indépendant de la charge, dérive sur des jours, se résorbe seul |
| Pente | asymétrie de résistance | proportionnelle au courant total — **le seul terme qui justifie de démonter une cosse** |

Ce qui se mesure est donc `I1 - I2` relevé **à plusieurs courants totaux** :
pente nulle, branches équilibrées ; pente non nulle, asymétrie réelle dont le
signe désigne le coupable. L'entité `Ecart courant packs Systeme` expose
directement cette grandeur.

Le pourcentage trompe : à écart de tension constant, l'écart en ampères ne
bouge pas quand la charge augmente, mais rapporté à un total plus grand il se
rapproche de 50 %. Un partage qui « s'améliore » à 40 A n'a rien prouvé.

Sans tracer de droite : un **échelon** de courant. Sur quelques secondes l'OCV
ne bouge pas, le terme constant s'annule, et `dI1 x R1 = dI2 x R2`.

**`Ecart tension packs` porte un offset de mesure de −52 mV**, établi le
12/09/2026 : à 0,0 A sur les deux packs, en parallèle sur la barre, donc au
même potentiel, l'entité lisait −47 à −58 mV toute une nuit. Les deux BMS ne
lisent pas pareil sur le même nœud ; le pack 2 affiche ~52 mV de plus. À
retrancher avant toute interprétation. Non vérifié : que l'offset soit
constant en tension — à relever de nouveau au repos vers 57 V.

## Le câblage est mesuré et innocenté — 12/09/2026

Une semaine de questions sur un disjoncteur défectueux ou une asymétrie de
câblage entre les deux batteries, réglée en une heure de charge à 40 A.

**La droite, d'abord.** Quatre paliers de consigne AC1 (10 / 20 / 30 / 40 A par
onduleur), `I1 − I2` moyenné sur les trois dernières minutes de chaque palier —
jamais une lecture isolée, la dispersion d'un échantillon est de ±0,5 A :

| I système | I1 − I2 |
|---|---|
| 20 A | +0,4 A |
| 40 A | +0,45 A |
| 60 A | −0,6 A |
| 80 A | −1,0 A |

Pente −0,025 A/A → R1 = 1,05 × R2. Ordonnée +1,1 A → OCV1 < OCV2. Le
quatrième point est tombé sur la droite prédite par les trois premiers.

**Le millivoltmètre ensuite**, à ~40 A dans chaque branche (`R = ΔV / I`, le
courant lu sur les BMS et les onduleurs à l'heure de chaque lecture) :

| Branche | ΔV (+) | ΔV (−) | R par polarité |
|---|---|---|---|
| Disjoncteurs, 8 pôles | 24 mV | 24 mV (un à 25) | 0,60 mΩ |
| Pack 1 → barre | 44 mV | 44 mV | 1,11 mΩ |
| Pack 2 → barre | 45 mV | 45 mV | 1,12 mΩ |
| Barre → Ond1 | 40 mV | 40 mV | 1,00 mΩ |
| Barre → Ond2 | 44 mV | 45 mV | 1,11 mΩ |
| Plot du pack → cosse (le boulon), à 60 A | < 2 mV | < 2 mV | < 0,03 mΩ |

Les deux branches batterie sont **identiques à 0,3 %** — 2,2 mΩ chacune, dont
1,2 pour la paire de pôles du disjoncteur. Les 5 % de la pente ne sont donc pas
dans le circuit : avec 10 à 20 mΩ de résistance interne par pack contre 2,2 mΩ
de câblage, ils correspondent à 0,5–1 mΩ d'écart interne entre deux lots de
cellules. Rien à resserrer, rien à changer.

Ce que ça referme : pas de disjoncteur défectueux (huit pôles à ±2 %) ; pas
d'asymétrie de câblage ; les croisements de courbes sur plusieurs jours sont le
terme d'écart de tension à vide, qui dérive avec les SOC — un câblage, fixe, ne
peut pas les produire.

## Première charge complète instrumentée — 12/09/2026

De 11:05 à 18:08, de 18 à 100 %, courant monté par paliers jusqu'à 120 A
système. Les deux compteurs ont touché le plein — c'est ce qui les
**recalibre**, un BMS n'intègre que des ampères-heures :

| | Pack 1 | Pack 2 |
|---|---|---|
| Restant / total | 325,24 / 325,87 Ah | 324,99 / 325,0 Ah |
| SOC | 99,8 % | 100,0 % |
| Température (départ 20,4 °C) | 30,3 °C | 30,8 °C |

**L'offset de tension est confirmé constant.** −51 mV à 54,44 V contre −52 mV
à 51,44 V le matin : trois volts plus haut, un millivolt d'écart. Ce n'est donc
pas un défaut de gain d'ADC. Ajouter **+0,051 V** à toute lecture de
`Ecart tension packs`. Lequel des deux BMS a raison reste inconnu — seul un
voltmètre sur la barre le dirait.

**Le delta cellules monte en fin de charge** : 8 mV à 35 % de SOC, **46 à
54 mV à 100 %**. Ce n'est pas une dégradation, c'est la courbe du LiFePO4 qui
se redresse au-dessus de 95 %. Mais l'équilibrage n'a probablement pas
travaillé : la cellule la plus haute a plafonné à 3,410 V (pack 1) et 3,418 V
(pack 2), la charge s'étant terminée sur la **tension de pack** (54,4 V,
consigne H35) et non sur une cellule. Ces BMS n'activent leurs résistances
qu'au-delà de 3,4–3,45 V par cellule.

À retenir : **terminer une charge à petit courant** (10–20 A système sur la
dernière heure) pour laisser l'équilibrage rattraper le delta. À vérifier au
prochain cycle.

## Le compteur de cycles ne compte pas des cycles

Il s'incrémente en **ampères-heures cumulés**, pas en charges complètes
validées. Constaté le 11/09/2026 : le pack 2 est passé de 3 à 4 pendant une
période sans aucune charge. La conclusion tirée le 3 septembre — « le pack 2
a refusé de valider une charge complète » — était donc fausse. C'est écrit
dans les deux fichiers de pack, au-dessus de l'entité.

## Surface exploitable du BMS

Établie par test le 24/08/2026, les deux formes de requête essayées :

```
42 analogique | 44 alarmes | 47 seuils | 51 versions   ... RÉPONDENT
4F | 83 | 92 | B0 ..................................... MUETS
```

Ces services n'existent pas dans ce firmware. Pas d'historique des
protections, pas de statistiques de capacité.

## Le défaut ÷10 du maître

Le BMS maître lit les données de l'esclave **divisées par dix** sur le lien de
parallélisme. Il déclare 32,5 Ah là où l'esclave affiche 325 Ah, et un total
système de ~358 Ah au lieu de 651.

L'erreur ne reste pas dans l'écran : elle sort sur le **CAN vers l'onduleur**.
Vérifié le 4 septembre par chronométrage d'une transition — le SOC transmis
suit le calcul ÷10 à dix secondes près, écartant les hypothèses « pack maître
seul » et « vrai système ».

**Résolu le 29/09/2026.** Après trois semaines d'échanges — démonstration
mesurée, vidéo, câblage innocenté — Docan a mis à jour le module WiFi du pack
1 à distance, puis le BMS maître est passé de V1.0.1 à V1.0.4, entre 21:44 et
22:09. Vérifié :

- l'écran du maître affiche une capacité système d'environ **651 Ah** au lieu
  de 358,4, et le courant de l'esclave tel que l'esclave l'affiche ;
- la limite de courant de charge transmise par CAN à l'onduleur (H34 côté
  Growatt) passe de **110 A à 210 A** : 100 A pour le maître plus 10 A pour
  l'esclave divisé par dix, devenus deux packs comptés normalement ;
- le réglage PYLON du RS485 est conservé, l'ESP lit les deux packs comme
  avant, aucune alarme.

Pendant la mise à jour, la liaison CAN est tombée deux fois (21:56 et 22:07)
et le binaire *[BMS] Communication Error* côté Growatt l'a signalé, deux
heures après sa correction. Le SOC transmis, lui, ne se départagera du modèle
« ÷ 10 » que lorsque les deux packs seront à des niveaux différents : à
99,8 % les deux calculs coïncident.

Au passage, la vidéo d'installation de Docan montrait le même défaut sur leur
propre banc — 303,1 / 344,6 Ah pour deux packs dont l'un contenait à lui seul
environ 292 Ah — ce qui a clos la discussion sur le câblage.

## L'alarme T2 est filtrée

L'octet d'alarme de la sonde 2 vaut `0x02` en permanence, sur les **deux**
packs, à 23 °C pour un seuil à 65 °C. Champ non implémenté, pas une surchauffe.

Le filtre porte sur la sonde 2 **et la valeur exacte `0x02`** : toute autre
valeur ressortirait. Le champ brut reste visible dans `Alarmes brut PackN`.

Sans ce filtre l'entité d'alarme ne dirait jamais « OK » et serait
inexploitable en automatisation.

## Ce qui n'est pas ici

`secrets.yaml` — WiFi, clé d'API, mot de passe OTA. Il vit uniquement sur le
serveur, et `.gitignore` en interdit l'ajout.

## Rafraîchir depuis le serveur

```
esphome_pull_files(filenames=["docan.yaml", "packages/dc-pack1.yaml", ...])
```

Le paramètre s'appelle `filenames`. Omis, l'outil ne rend que la racine.

## Taille des trames et tampon de réception

```
trame = 18 octets d'ossature + INFO

analogique  INFO = 30 + 4 x ncell + 4 x ntemp = 114   ->  132 octets
alarmes     INFO = 14 + 2 x ncell + 2 x ntemp =  56   ->   74 octets
seuils      INFO >= 50                                ->  ~68 octets
```

`rx_buffer_size: 512` est posé explicitement dans `docan.yaml` : 132 sur 512,
26 % occupés. Ce n'était pas une question de place — les 256 octets par défaut
d'ESPHome suffisaient, y compris pour un hypothétique pack de 24 cellules
(164 octets). La ligne supprime la dépendance à un défaut amont non écrit.

Coût mesuré : nul à la compilation, `.bss` identique de part et d'autre du
changement. Le pilote UART de l'ESP-IDF alloue ce tampon sur le tas.

## Version ESPHome

Compilé et flashé avec **2026.9.0** depuis le 26/09/2026 (2026.8.2 du 12 au
26/09). Aucun composant `modbus` ici : les dépréciations de
`command_throttle` puis de `skip_updates` ne concernent que le montage
Growatt. La 2026.9.0 n'a rien signalé sur ce fichier, hormis l'OTA.

## 26/09/2026 — OTA chiffré

Le mot de passe OTA coûtait 3,5 Ko de flash et ne servait qu'aux flasheurs
sans chiffrement ; ESPHome retire le repli en clair en 2027.3.0. Le bloc
`ota:` utilise désormais `encryption:` avec la clé API, comme sur les sept
autres appareils du serveur. Premier flash à 22:51 encore en clair, le
second à 22:54 annonce « Encrypted connection established »,
`config_hash=0x958f0561`. Aucune entité touchée. Reste un avertissement de
compilation : le point `/update` du `web_server` accepte toujours une image
en clair.

## 03/10/2026 — partage des packs, réglages des BMS, seuils de température

**Une décharge de 20 h à 2,3 A, puis des paliers à 20–55 A.** Du 01/10 18:20
au 02/10 14:14, onduleurs en SBU, le pack 1 a fourni 31,0 Ah et le pack 2
14,4 Ah (68 / 32 %). Le partage a convergé seul — 78 % à 20:20, 70 % à 02:20,
61 % à 08:20, 57 % à 12:20 — pendant que l'écart de tension corrigé restait
entre +1 et +10 mV. Le 03/10, sous charge :

| Courant total | Pack 1 | Pack 2 | Part du pack 1 |
|---|---|---|---|
| 31 A | −15,6 A | −15,4 A | 50 % |
| 29 A | −14,4 A | −14,4 A | 50 % |
| 22 A | −10,7 A | −11,5 A | 48 % |
| 47 A | −21,9 A | −24,7 A | 47 % |
| 55 A | −26,1 A | −28,9 A | 47 % |

C'est la droite du 12/09 : pente de quelques pour cent (R1 ≈ 1,05 × R2) et
terme constant qui a changé de signe entre le 02 et le 03/10, le pack 1
ayant pris de l'avance. À faible courant le terme constant domine et le
pourcentage n'a aucun sens ; à fort courant on est à 50 / 50. Rien à
resserrer, rien à régler.

**Aucun réglage de BMS ne répartit le courant entre deux packs**, et les
réglages sont identiques. Relevé du 03/10 dans l'application iPhone *BMS
Insight*, 36 paramètres, les mêmes sur les deux packs après la mise à jour
V1.0.4 du pack 1 :

- tensions : cellule 3 650 / 2 700 mV (protection), 3 580 / 2 800 mV
  (alarme) ; total 58,4 / 43,2 V et 57 / 44,8 V ; écart 800 et 500 mV ;
- courants : protection 220 A, alarme 205 A, en charge comme en décharge ;
- températures (protection) : charge 55 / −5 °C, décharge 60 / −20 °C,
  ambiance 65 / −25 °C, MOS 110 °C ;
- fin de charge : *Constant V* 56,00 V, *Constant C* 2 A ; équilibrage à
  partir de 3 450 mV et 30 mV d'écart ; alarme SOC 15 %, retour 20 % ;
- options : limiteur « Passive current limit 100 % », protection incendie,
  shunt et antivol fermés.

Firmware `DR_YP02_16S200JC26_V1.0.4_T1`, module `DR-WIFI02_V1.4.11`, boot
V2.21 des deux côtés. Deux lots de fabrication : 03/09/2025 et 17/11/2025.

**Une seule différence : *Design Capacity* vaut 300 Ah sur le pack 1 (maître)
et 314 Ah sur le pack 2.** Sans effet sur le SOC (restant ÷ pleine) ni sur le
partage ; question posée à Docan le 03/10 (réponse attendue), rien n'a été
écrit dans les BMS.

**Qui est qui dans l'application** — l'ordre y est l'inverse du nôtre :

| Module WiFi | Écran du pack | Ici et dans HA |
|---|---|---|
| `DWF841FE8118BD1` | `#.00` | Pack 1, maître |
| `DWF9454C538CBA9` | `#.01` | Pack 2, esclave |

Établi par la capacité restante lue au même instant des deux côtés.

**Les seuils de température lus par l'ESP sont 10 °C trop hauts.** Le
service 47 donne 65 / 5 °C en charge et 70 / −10 °C en décharge ; les BMS
sont réglés à 55 / −5 et 60 / −20. Les températures *mesurées*, décodées avec
la même formule, tombent juste (20,6 °C pour 21 / 21 / 20 / 20 à l'écran) :
c'est donc la trame du BMS qui est décalée, pas le décodage — même famille
que la sonde recopiée cinq fois et l'alarme T2. Les entités ne sont pas
corrigées ; la référence est l'application. Deux noms sont trompeurs et le
restent, on ne renomme pas une entité : *Seuil cellule basse* (3,58 V) est
l'alarme de surtension cellule, *Seuil cellule sous-tension* (2,80 V)
l'alarme de sous-tension, la coupure étant à 2,70 V.

L'écran des packs montre toujours ce que le bus ne donne pas : MOS 23 °C et
ambiance 25 °C quand les cellules sont à 20–21 °C.

## 04/10/2026 — capacité du pack 1 passée à 200 Ah, trames abîmées — EN COURS

**Ce qui a changé dans le BMS du maître**, entre 18:10 et 18:25, sans
qu'on sache encore par qui (question posée à l'utilisateur et à Docan) :

| Paramètre du pack 1 | 03/10 | 04/10, 19:34 |
|---|---|---|
| *Full Capacity* | 325,8 Ah | **200 Ah** |
| *Design Capacity* | 300 Ah | **200 Ah** |
| *Remaining Capacity* | 325,7 Ah | 199,8 Ah (SOC conservé, 99,9 %) |
| Firmware, module, boot | V1.0.4_T1 / V1.4.11 / V2.21 | inchangés |
| Seuils de protection, cycles (6) | | inchangés |

Le pack 2 n'a pas bougé (325 / 314 Ah). Le système annonce 525 Ah au lieu
de 651. Cellules à 3,353–3,354 V, 1 mV d'écart, aucune alarme : le pack est
sain, c'est son compteur qui est faux. En décharge, son SOC descendra 1,6
fois trop vite.

Pendant ces quinze minutes le pack 1 n'a répondu à rien, et l'ESP a
redémarré huit fois (l'uptime retombe à zéro à chaque fois ; cause non
établie). Côté onduleurs, à 18:20 : perte de la liaison BMS, puis *Max
Charge Current* à 400 A sur Ond1 et 1 000 A sur Ond2, restés ainsi.

**Qui est qui, complété** : la page *PACK information* du module `BD1`
(pack 1, maître) donne PACK SN 9141048225090032, BMS SN DJM2508290045,
fabrication 03/09/2025. Le pack 2 est donc le 9141052925110027 du 17/11/2025.

**À partir de 19:36, les réponses du pack 1 arrivent abîmées** sur le
RS485 — celles du pack 2 restent propres, sur la même paire. Caractères
hors hexadécimal (`<`, `?`, `>`, octets au-dessus de 0x7F), trames d'un
caractère trop courtes, et un compteur de cycles lu à 0 au lieu de 6.

**Le défaut que cela a révélé chez nous.** Aucun parseur ne vérifiait la
somme de contrôle. Une trame d'alarme abîmée est partie telle quelle dans
*Alarmes brut Pack1* ; Home Assistant refuse une chaîne qui n'est pas de
l'UTF-8 et coupe la liaison API, se reconnecte, reçoit la même chaîne :
trois reconnexions par seconde de 19:37 à 19:47, 108 entités qui
clignotent, autant de lignes dans la base.

Corrigé le soir même dans les six lectures (`analog`, `alarme`, `seuils`,
deux packs) : une trame qui contient autre chose que de l'hexadécimal est
rejetée, et la somme de contrôle PYLON est exigée **dès qu'elle a été vue
juste une fois** depuis le démarrage (`p1_chk`, `p2_chk`) — on ne parie pas
sur la conformité d'un BMS qui remplit mal ses trames. Vérifié après le
flash : liaison API stable, pack 2 lu normalement, pack 1 rejeté à chaque
cycle puis « Pas de réponse » au troisième, ce qui est l'affichage voulu.

Rien n'a été écrit dans les BMS.
