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
| Pack 1 | adresse PYLON 01, **maître**, DIP tous OFF, firmware BMS V1.0.1 |
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

Signalé à Docan, mise à jour du firmware maître en cours de discussion.

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

## À faire au prochain flash

Compilé avec ESPHome **2026.7.4**. L'add-on est passé en **2026.8.2**, qui
déprécie `command_throttle` au profit de `turnaround_time` sur le composant
`modbus`.
