# NEON ESCAPE

**Un jeu de survie 2D contrôlé par la webcam, sans clavier ni souris pendant la partie.**

Projet collectif de quatre étudiants en MP2I, associant programmation Python, vision par ordinateur, mathématiques et développement de jeux vidéo.

## 1. Concept du jeu

Le joueur déplace un objet vert devant la webcam pour contrôler un personnage librement dans les deux dimensions. Il doit récupérer des pièces dorées tout en évitant des obstacles rouges qui descendent depuis le haut de l'écran.

- Chaque pièce récupérée rapporte un point et déclenche l'apparition d'une nouvelle pièce.
- Tous les 10 points, le niveau augmente et les obstacles deviennent plus rapides et plus fréquents, jusqu'aux limites programmées.
- Une collision avec un obstacle termine la partie.
- Le jeu n'a pas de fin prédéfinie : l'objectif est de battre son record.
- Si l'objet vert n'est plus détecté, la simulation se met en pause.
- Le meilleur score est sauvegardé dans un fichier texte lorsque le record est battu à la fin d'une partie.

## 2. Installation

### Prérequis

- Python 3 et pip (environnement utilisé pour le développement : Python 3.14.7 sous Windows).
- Une webcam accessible et un objet vert bien visible, de préférence mat.
- Un environnement graphique permettant d'ouvrir les fenêtres OpenCV et Pygame.

Dans le terminal de VS Code, installer les dépendances :

```bash
python -m pip install opencv-python==4.13.0.90 numpy pygame-ce
```

La version d'OpenCV indiquée est celle qui a fonctionné sur le poste de développement. `pygame-ce` s'importe sous le nom `pygame` ; il n'est pas nécessaire d'installer Pygame classique en plus.

Vérifier les installations :

```bash
python --version
python -c "import cv2, numpy, pygame; print(cv2.__version__, numpy.__version__, pygame.version.ver)"
```

Les modules `random`, `math` et `pathlib` appartiennent à la bibliothèque standard de Python et ne nécessitent aucune installation supplémentaire.

## 3. Lancement et commandes

Placer les cinq fichiers Python dans un même dossier, ouvrir le terminal dans ce dossier et exécuter :

```bash
python main.py
```

| Commande | Action |
|---|---|
| Déplacer l'objet vert | Déplacer le personnage horizontalement et verticalement |
| Retirer l'objet du champ de la webcam | Mettre temporairement la simulation en pause |
| R (fenêtre Pygame, après une défaite) | Rejouer |
| Q ou Échap (fenêtre Pygame) | Quitter |
| Q (fenêtre OpenCV active) | Quitter en mode démonstration |
| Croix de la fenêtre Pygame | Fermer le jeu |

Pour une bonne détection, utiliser un éclairage régulier et éviter les autres objets verts dans le champ de la caméra.

## 4. Architecture du projet

```text
NEON_ESCAPE/
├── main.py         # Coordination des modules et boucle principale
├── tracking.py     # Vision par ordinateur et lissage
├── game.py         # Fenêtre, personnage et affichage graphique
├── obstacles.py    # Obstacles, collisions et difficulté
├── scoring.py      # Pièces, score, niveaux et record
├── highscore.txt   # Créé automatiquement lors de la sauvegarde d'un record
└── README.md       # Documentation du projet
```

Le projet repose sur une architecture modulaire : chaque fichier possède une responsabilité précise. `main.py` importe les fonctions des autres modules et synchronise leur exécution au sein d'une seule boucle principale.

| Membre | Module | Responsabilité |
|---|---|---|
| 1 | `tracking.py` | Capture vidéo, détection de couleur, coordonnées et lissage |
| 2 | `game.py` | Moteur graphique, déplacement du joueur et limites du terrain |
| 3 | `obstacles.py` | Génération, mouvement, collisions et difficulté des obstacles |
| 4 | `scoring.py` | Pièces, score, niveaux, sauvegarde du record et écran Game Over |
| Groupe | `main.py` | Assemblage et validation collective |

## 5. Fonctionnement scientifique et algorithmique

### Suivi par couleur

Le module `tracking.py` réalise les opérations suivantes sur chaque image :

1. Capture de l'image de la webcam et effet miroir.
2. Conversion de l'espace colorimétrique BGR vers HSV.
3. Création d'un masque binaire avec `cv2.inRange()`.
4. Extraction des contours extérieurs avec `cv2.findContours()`.
5. Sélection du plus grand contour dont l'aire dépasse 300 pixels carrés.
6. Calcul du rectangle englobant et de son centre.
7. Lissage exponentiel des coordonnées.
8. Adaptation des coordonnées aux dimensions de la fenêtre Pygame (800 × 600 pixels).

Les seuils HSV utilisés sont `[40, 50, 50]` et `[80, 255, 255]`. Ils peuvent être ajustés selon l'objet et l'éclairage.

### Lissage exponentiel

Pour réduire les fluctuations de la position détectée, nous utilisons une moyenne pondérée entre l'ancienne position lissée et la nouvelle mesure :

```math
p_{n+1} = (1-\alpha)p_n + \alpha P_n
```

Cette relation est appliquée aux deux coordonnées, avec un coefficient initial de 0,25. Un coefficient plus faible réduit davantage les fluctuations, mais augmente le retard du déplacement.

### Déplacement et collisions

Les obstacles sont des rectangles ou des cercles générés aléatoirement. Leur déplacement vertical est calculé en fonction du temps écoulé :

```math
y_{n+1} = y_n + v\Delta t
```

Les positions décimales sont conservées entre les images pour limiter les erreurs d'arrondi. Les collisions rectangulaires utilisent `colliderect()`. Pour les obstacles circulaires, le programme calcule la distance entre le centre du cercle et le point du rectangle du joueur le plus proche : il y a collision si cette distance est inférieure ou égale au rayon.

### Progression de la difficulté

Le niveau est calculé à partir du score :

```math
N = 1 + \left\lfloor \frac{S}{10} \right\rfloor
```

La vitesse et l'intervalle d'apparition des obstacles sont définis par :

```math
v_N = \min(180[1+0{,}2(N-1)],450)
```

```math
T_N = \max(1{,}4-0{,}12(N-1),0{,}45)
```

La vitesse est exprimée en pixels par seconde et l'intervalle en secondes. La difficulté atteint son plafond au niveau 9.

| Niveau | Score | Vitesse (px/s) | Intervalle (s) |
|---|---|---:|---:|
| 1 | 0–9 | 180 | 1,40 |
| 2 | 10–19 | 216 | 1,28 |
| 3 | 20–29 | 252 | 1,16 |
| 5 | 40–49 | 324 | 0,92 |
| 9 et plus | 80 et plus | 450 | 0,45 |

## 6. Mode démonstration

Dans `main.py`, le paramètre suivant permet de présenter les étapes du traitement d'images :

```python
DEMONSTRATION = True
```

Quatre fenêtres supplémentaires sont alors affichées :

| Fenêtre | Contenu |
|---|---|
| Webcam BGR | Image de la webcam après effet miroir |
| Webcam HSV | Matrice HSV affichée directement ; ses couleurs ne correspondent pas à un rendu naturel |
| Masque vert | Pixels sélectionnés en blanc, autres pixels en noir |
| Tracking | Contours, rectangle englobant, position brute (rouge), position lissée (bleu) et coordonnées du jeu |

Pour jouer sans ces fenêtres, définir `DEMONSTRATION = False`.

## 7. Technologies utilisées

| Technologie | Utilisation |
|---|---|
| Python | Langage principal |
| OpenCV | Capture et traitement des images |
| NumPy | Représentation des images et seuils colorimétriques |
| Pygame CE | Affichage, événements, objets graphiques et collisions |
| Pathlib | Lecture et sauvegarde du record |
| Random | Génération aléatoire des pièces et obstacles |
| Math | Calcul de distances euclidiennes |

## 8. Tests et dépannage

Les modules de vision et de moteur graphique disposent de tests indépendants :

```bash
python tracking.py
python game.py
```

Le second permet de tester le déplacement avec les flèches du clavier. Le test complet s'effectue avec `python main.py`.

- **Module introuvable :** vérifier l'installation des bibliothèques et l'interpréteur sélectionné dans VS Code. Vérifier aussi que les cinq fichiers Python se trouvent dans le même dossier.
- **Webcam inaccessible :** fermer les autres applications utilisant la caméra et vérifier les autorisations Windows.
- **Objet non détecté :** vérifier l'éclairage, le masque binaire, la taille de l'objet et les seuils HSV.
- **Erreur de chargement de `cv2.pyd` :** sur le poste de développement, l'installation de la version officielle d'OpenCV 4.13.0.90 a résolu le problème. Si le blocage persiste, demander de l'aide au support informatique sans désactiver les protections de Windows.

## 9. Limites et perspectives

NEON ESCAPE utilise des méthodes de vision par ordinateur classique, sans modèle entraîné ni réseau neuronal. Le suivi suppose que l'objet contrôlé correspond au plus grand contour vert admissible. La détection dépend donc de l'éclairage et de l'environnement.

Les pièces peuvent apparaître sur un obstacle, certaines collisions sont approximées par des rectangles et la difficulté plafonne au niveau 9. Le record est sauvegardé à la défaite, et non lors d'une fermeture en cours de partie. Les performances et la fluidité dépendent également de la webcam et de l'ordinateur.

Des améliorations futures pourraient porter sur la calibration automatique de la couleur, la génération de pièces hors des obstacles et la reconnaissance des gestes sans objet coloré.

## 10. Objectif pédagogique

Ce projet permet de mettre en pratique le traitement matriciel des images, la géométrie, le filtrage numérique, les probabilités, la programmation modulaire et la coordination d'un travail collectif autour d'une application interactive.

