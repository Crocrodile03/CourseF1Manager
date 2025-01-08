# CourseF1Manager

## Usage
Le programme est écrit en C et m'utilise que les librairies standars.
Commande pour compiler le programme: `gcc course.c -o course`

Pour fonctionner, le programme a besoin de 2 fichiers: `drivers.csv` et `tracks.csv`.
Commande pour démarrer le programme: `./course`

Après une confirmation de l'utilisateur, le programme va simuler une phase de week-end (essai libre, qualification ou course), affiche les résultats et les enregister dans un ou plusieurs fichiers.  Au démarrage, il lira les fichiers déjà créés pour savoir où en est l'avancement du week-end et le classement des pilotes.

### drivers.csv
Le fichier contient 3 champs: le numéro du pilote, 3 lettres majuscules (le début du nom de famille) et le prénom et nom de famille complet
Le fichier doit contenir 20 lignes pour les 20 pilotes inscrits au championnat.
Exemple:
```
1;VER;Max Verstappen
11;PER;Sergio Pérez
44;HAM;Lewis Hamilton
```

### tracks.csv
Le fichier contient 4 champs: le pays, le nom du circuit, le type de week-end et la distance en mètre d'un tour de circuit.
Le type de week-end, peut-être soit "normal" soit "sprint".  Un week-end "normal" est composé de 3 essais libres, 3 séances de qualification et la course.
Un week-end "sprint" est composé d'un essai libre, 3 séances de qualification pour le sprint, une course "sprint", puis 3 séances de qualification classique et la course.
Exemple:
```
Bahrain;Bahrain International Circuit;normal;5412
Arabie Saoudite;Jeddah Corniche Circuit;normal;6174
Australie;Albert Park Circuit;normal;5278
Azerbaijan;Baku City Circuit;sprint;6003
```
### les fichiers créés par le programme
Le programme va créé plusieurs fichiers lors des différentes exécutions, on peut les classer en 3 types:
* championship.txt: c'est le fichier qui indique la dernière course/phase exécuté.  Il contient 2 lignes: `race=n` et `phase=m`.  Au démarrage, le programme lit le fichier pour savoir quelle sera la phase suivante à simuler.  Si le fichier n'existe pas, on suppose que l'on est au début du championnat
* race_nn_pp.csv: c'est le résultat de la simulation.  On retouve la liste des pilotes classés en fonction de leur résultat.  Il sera utilisé lors de certaines pour déterminer le classement des pilotes sur la piste de départ (qualification 2/3, sprint et course finale).  Il contient le numéro du pilote, ses temps pour le meilleur tour et meilleur temps de chaque section.
* race_nn_(race|sprint)_ranking.csv: c'est le résultat du sprint ou de la course.  On retrouve 2 informations: le numéro du pilote et le nombre de point marqué.  Ces fichiers sont lu par le programme à la fin de la simulation pour afficher le classement des pilotes.
 
## Description du programme
Ce programme simule un week-end de course de formule 1: à savoir les essais libres et qualifications ou la course (ou sprint).
Pour ce faire, le programme principal va démarrer 3 types de sous-process (controller, carSimulator, screenManager) et attendre qu'ils se terminent.
Le premier sous-process est le `controller`, il est le composant central.  Il traite les données envoyées par les carSimulators et les cumulent pour le screen manager.
Le screenManager affichent à l'écran les données compilées par le controller.  Le carSimulator simule une voiture sur le circuit.  Le programme en démarre autant que de pilote en course (entre 10 et 20 en fonction de la phase du week-end).

### Fonctionnement interne
Les sous-process sont démarrés via des `fork()`.  Au plus fort de l'exécution, on aura 23 processus (le programme "main", le controller, le screenManager et jusqu'à 20 carSimulator).  Ces processus communiquent leurs données via une mémoire partagée (créé par le "main" et récupéré par tous les fils).
Afin de garantir un accès cohérent aux données, l'algorithme "Courtois" a été implémenté.  Toutes les accès à la mémoire partagée en mode "concurrent" sont dans des fonctions bien définies

Liste des fonctions qui "lisent" des données
| fonction                                                             | Description |
| :------------------------------------------------------------------- | :----------------- |
| void readSharedMemoryData(void* data, enum SharedMemoryDataType smdt)| Retourne dans la zone `data` l'information demande, cela peut être `CAR_STATS`: les données cumulées des pilotes (total time, distance, best lap, ...), `RUNNING_CARS`: le nombre de pilote encore en course, `CAR_TIME_AND_STATUSES`: les données pour une section (c'est la zone mémoire utilisée par les carSimulator pour envoyer les données aux controller), `RACE_OVER`: un flag qui indique si le course est terminée, ´PROCESSED_FLAGS´: un tableau avec tous les flags "processed" des structures CAR_TIME_AND_STATUS.   |
| int getRunningCars()| Utilise `readSharedMemoryData` pour retourner directement le nombre de pilote encore en course |
| bool isRaceOver()| Utilise `readSharedMemoryData` pour retourner directement le flag RaceOver |

Liste des fonctions qui "écrivent" des données
| fonction                                                              | Description |
| :-------------------------------------------------------------------- | :----------------- |
| void decrementRunningCars()| Décrémente le compteur de pilote encore en course | 
| void setRaceAsOver()| Met à `true` le flag raceOver | 
| void setCarTimeAndStatusAsProcessed(int i)| Met à `true` le flag processed de la zone CarTimeAndStatus n° i | 
| void updateCarStat(CarStat carStat, int i)| Met à jour les données CarStat de la zone n°i | 
| int sendDataToController(int id, CarTimeAndStatus status)| Met à jour les données d'une section pour le controller.  Retourne le nombre de milliseconds "perdues" dans l'opération (c'est la somme des waits que la fonction à fait en attendant d'avoir accès en exclusive à la mémoire partagée | 


