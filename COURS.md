# Cours 4 — Escrime : de la vidéo à une API

> **Module de cours produit par Claude Code (Anthropic), modèle Claude Fable 5.2. Créé le 25/11/2026**, à partir des sources officielles listées en fin de document. Le cours donne le chemin ; le code du projet est écrit par Charles Burgy.

| | |
|---|---|
| **Durée estimée** | 4 semaines, soit 16 à 20 séances de 1 à 2 h |
| **Niveau** | Intermédiaire |
| **Coût** | 0 €. Tout tourne en local, sur le processeur |
| **Avant de commencer** | Un téléphone pour filmer, un pied ou un appui stable |

## Le projet en une phrase

Une API reçoit une courte vidéo d'un tireur filmé de profil et renvoie les actions reconnues (marche, retraite, fente) avec quelques mesures, puis un court compte rendu rédigé à partir de ces seules mesures.

## Pourquoi ce projet

Il sort du « tout modèle de langage ». Il montre la chaîne complète d'un produit d'IA : des données que tu as créées, un modèle classique entraîné et évalué proprement, un service web testé, et des tests lancés automatiquement. Le modèle de langage n'arrive qu'à la fin, pour rédiger, et il est contrôlé.

## Ce que tu sauras faire à la fin

- Constituer et annoter un petit jeu de données, avec l'accord des personnes filmées.
- Extraire la pose d'un corps dans une vidéo.
- Construire des mesures à partir de points du corps.
- Entraîner et évaluer un classifieur sans fuite de données.
- Servir le tout par une API web, avec sa documentation et ses tests.
- Faire tourner les tests automatiquement sur GitHub.

## L'arborescence visée

```
04-escrime-video/
├── escrime_video/
│   ├── video.py         lecture des images
│   ├── pose.py          image -> points du corps
│   ├── mesures.py       points -> angles, distances, vitesses
│   ├── classifieur.py   entraînement et prédiction
│   ├── compte_rendu.py  mesures -> texte (modèle local)
│   ├── api.py           le service FastAPI
│   └── schemas.py
├── tests/
├── data/
│   ├── annotations.csv
│   └── clips/           les vidéos (ignorées par git)
├── modeles/             le classifieur entraîné
├── eval/
└── .github/workflows/   les tests automatiques
```

---

## Séance 0 — Mise en place

1. MediaPipe accepte Python 3.9 à 3.14 sur Windows (relevé du 25/11/2026) : Python 3.12 convient, rien à changer.
2. `uv sync`, `.env`, dépôt git local.
3. Vérifier que `data/clips/` est bien dans `.gitignore`.

---

## Module 1 — Les données et l'accord des personnes

**Objectif** : 60 à 100 clips courts, annotés, obtenus proprement.

**Notions**

- Une vidéo d'une personne est une donnée personnelle. Il faut son accord, donné en connaissance de cause : à quoi servent les images, où elles sont rangées, qu'elles ne seront pas publiées.
- Les clips restent sur ton PC. Le dépôt GitHub ne contient que les annotations, les mesures (des nombres, sans image) et la méthode pour refaire le jeu.
- Une annotation est l'étiquette donnée par un humain. La qualité du modèle ne dépassera pas celle des annotations.

**À faire**

1. Écrire un protocole de tournage en 10 lignes : caméra fixe, de profil, à hauteur de hanche, tireur entier dans l'image, bon éclairage.
2. Filmer, toi d'abord. Si d'autres tireurs participent, garder leur accord écrit et ne publier aucune image d'eux.
3. Couper en clips de 2 à 4 secondes, une action par clip : `marche`, `retraite`, `fente`, `garde` (rien ne se passe).
4. Remplir `data/annotations.csv` : fichier, action, tireur (un code, pas un nom), main armée, date de tournage.
5. Viser un nombre de clips proche pour chaque action.

**Réussi quand** : le CSV décrit tous les clips, et quelqu'un d'autre pourrait refaire le tournage avec ton protocole.

**Piège** : si tous les clips viennent d'une seule séance, le modèle apprend la salle et la tenue. Filmer sur au moins deux séances.

---

## Module 2 — Lire une vidéo

**Objectif** : parcourir une vidéo image par image.

**Notions**

- Une vidéo est une suite d'images, à une cadence donnée (images par seconde).
- OpenCV lit les vidéos et rend chaque image sous forme de tableau numpy.
- OpenCV range les couleurs dans l'ordre bleu-vert-rouge. Beaucoup d'autres outils attendent rouge-vert-bleu : il faut convertir.

**À faire**

1. `uv add opencv-python`.
2. Écrire `video.py` : une fonction qui rend les images d'un clip une par une, avec leur instant en secondes.
3. Afficher pour un clip : largeur, hauteur, cadence, nombre d'images, durée.
4. Enregistrer une image sur dix dans un dossier, pour vérifier à l'œil.

**Réussi quand** : la durée calculée correspond à celle affichée par ton lecteur vidéo.

---

## Module 3 — La pose

**Objectif** : pour chaque image, les points du corps du tireur.

**Notions**

- L'estimation de pose trouve la position des articulations dans une image : épaules, coudes, poignets, hanches, genoux, chevilles.
- MediaPipe fournit un modèle déjà entraîné, qui tourne sur un processeur ordinaire.
- Chaque point vient avec un indice de visibilité. Un point caché ou douteux ne doit pas être utilisé tel quel.

**À faire**

1. Suivre le guide officiel du « Pose Landmarker » en Python (source en fin de document).
2. Écrire `pose.py` : un clip en entrée, un tableau en sortie (une ligne par image, les coordonnées de chaque point).
3. Dessiner les points sur la vidéo et la regarder. Où la pose se trompe-t-elle ?
4. Enregistrer les points de chaque clip dans un fichier, pour ne pas recalculer.
5. Mesurer : combien d'images traitées par seconde sur ton PC ?

**Réussi quand** : sur 5 clips, le squelette dessiné suit bien le tireur.

**Piège** : avec deux tireurs dans l'image, le modèle peut sauter de l'un à l'autre. D'où le protocole : un seul tireur par clip.

---

## Module 4 — Des points aux mesures

**Objectif** : des nombres qui ont un sens pour un escrimeur.

**Notions**

- Une mesure (« feature ») est un nombre calculé à partir des données brutes, plus parlant qu'elles : un angle de genou en dit plus que douze coordonnées.
- Normaliser : diviser les distances par une longueur du corps (par exemple hanche-épaule), pour ne pas dépendre de la distance à la caméra ni de la taille du tireur.
- Lisser : les points tremblent d'une image à l'autre. Une moyenne glissante calme le signal avant de calculer une vitesse.

**À faire**

1. Écrire `mesures.py`, avec au moins :
   - l'angle du genou avant et celui du genou arrière ;
   - l'écart entre les pieds, normalisé ;
   - la position horizontale du bassin, et sa vitesse ;
   - l'extension du bras armé (angle du coude) ;
   - la vitesse de la main armée.
2. Tester les calculs d'angle avec pytest, sur des points placés à la main (un angle droit, un bras tendu).
3. Tracer ces mesures dans le temps pour une marche, une retraite et une fente (matplotlib). Les trois actions se distinguent-elles à l'œil ?

**Réussi quand** : tu peux montrer un graphique où la fente se reconnaît sans regarder la vidéo.

---

## Module 5 — Le classifieur, sans fuite de données

**Objectif** : un modèle qui reconnaît l'action d'un clip qu'il n'a jamais vu.

**Notions**

- Apprentissage supervisé : on donne des exemples avec leur étiquette, le modèle apprend à prédire l'étiquette.
- Résumer un clip par un vecteur : le minimum, le maximum, la moyenne et l'amplitude de chaque mesure.
- Séparer entraînement et test **par clip**, et mieux, par séance de tournage ou par tireur. Sinon le score est faussé (guide 04, section 3). scikit-learn a des outils pour ça, les séparations « par groupes ».
- Commencer simple : une forêt aléatoire ou une régression logistique. Pas de réseau de neurones pour 80 clips.

**À faire**

1. `uv add scikit-learn pandas`.
2. Construire le tableau : une ligne par clip, les mesures résumées, l'action.
3. Écrire d'abord la référence la plus bête : toujours prédire l'action la plus fréquente. Quel score obtient-elle ?
4. Entraîner un classifieur, avec une graine fixe.
5. Évaluer par validation croisée par groupes.
6. Enregistrer le modèle entraîné dans `modeles/`, avec la date et la liste des mesures utilisées.

**Réussi quand** : ton classifieur bat nettement la référence bête, sur des clips jamais vus.

---

## Module 6 — L'évaluation

**À faire**

1. Sortir la précision et le rappel par action, et la matrice de confusion (guide 04, section 2).
2. Regarder chaque clip mal classé. La faute vient-elle de la pose, des mesures, de l'annotation ou du modèle ?
3. Regarder quelles mesures comptent le plus pour le modèle. Est-ce cohérent avec ce que tu sais de l'escrime ?
4. Tester sur un clip tourné dans d'autres conditions (autre salle, autre angle). Le score tient-il ? L'écrire tel quel.

| Action | Précision | Rappel | Nombre de clips |
|---|---|---|---|
| Marche | | | |
| Retraite | | | |
| Fente | | | |
| Garde | | | |

**Réussi quand** : le tableau et la matrice de confusion sont dans le README, avec les limites constatées.

---

## Module 7 — L'API

**Objectif** : n'importe quel programme peut envoyer une vidéo et recevoir le résultat.

**Notions**

- Jusqu'ici tu appelais les API des autres. Ici tu écris la tienne.
- FastAPI : on écrit une fonction Python, on décrit ses entrées et ses sorties avec Pydantic, et l'on obtient un service web avec sa documentation interactive (page `/docs`).
- Un point d'entrée (« endpoint ») : une adresse et une méthode, par exemple `POST /analyse`.
- Le modèle se charge une fois au démarrage du service, pas à chaque demande.

**À faire**

1. `uv add "fastapi[standard]"`.
2. Écrire `api.py` avec trois points d'entrée :
   - `GET /sante` : dit que le service tourne et quelle version du modèle est chargée ;
   - `POST /analyse` : reçoit un clip, rend l'action, la confiance et les mesures ;
   - `POST /compte-rendu` : reçoit les mesures, rend un texte (module 9).
3. Décrire les réponses dans `schemas.py`.
4. Refuser proprement : fichier trop gros, format non reconnu, aucun tireur détecté. Chaque refus a son code de statut et un message clair.
5. Lancer le service et l'essayer depuis la page `/docs`, puis depuis un script Python avec httpx.

**Réussi quand** : un script de 10 lignes envoie un clip et affiche l'action reconnue.

---

## Module 8 — Tests et tests automatiques

**Objectif** : chaque `git push` vérifie que rien n'est cassé.

**Notions**

- FastAPI fournit un client de test : on appelle l'API dans pytest sans lancer de serveur.
- L'intégration continue : un service (ici GitHub Actions, gratuit pour un dépôt public) relance les tests à chaque envoi de code.
- Les tests ne doivent pas dépendre des vidéos, qui ne sont pas sur GitHub. On teste avec des points du corps enregistrés, ou un clip de synthèse de quelques images.

**À faire**

1. Écrire les tests de l'API : la santé, une analyse réussie, chaque cas de refus.
2. Écrire `.github/workflows/tests.yml` en suivant le guide officiel pour Python.
3. Ajouter `ruff` pour vérifier le style du code, en local et dans le workflow.
4. Mettre le badge de résultat des tests en haut du README.

**Réussi quand** : la coche verte apparaît sur GitHub après un `git push`.

---

## Module 9 — Le compte rendu, rédigé puis vérifié

**Objectif** : cinq lignes de retour pour le tireur, sans un chiffre inventé.

**À faire**

1. Écrire `compte_rendu.py` : il reçoit le JSON des mesures et demande à un modèle local un retour court, en sortie structurée (le texte, et la liste des chiffres cités).
2. Contrôler par le code : chaque chiffre cité existe dans le JSON, à l'arrondi près. Sinon, refaire une fois, puis rendre un compte rendu sans chiffres.
3. Réutiliser `llm.py`, le journal et le cache des projets précédents.
4. Écrire dans le README ce que ce compte rendu est (un résumé de mesures) et ce qu'il n'est pas (un avis de maître d'armes).

**Réussi quand** : sur 10 clips, aucun compte rendu ne contient un chiffre absent des mesures.

---

## Module 10 — Docker (option) et finition

**Notions**

- Docker range le service et tout ce dont il a besoin dans une « image », qui tourne de la même façon sur toute machine.
- Docker Desktop demande une installation lourde sur Windows. Ce module est une option : le projet est complet sans lui. Vérifier les conditions d'usage dans le guide 01.

**À faire**

1. Si tu fais l'option : écrire un `Dockerfile`, construire l'image, lancer le service, appeler `/sante`.
2. Le schéma en Mermaid : vidéo, pose, mesures, classifieur, API, compte rendu.
3. Le README complet, avec la section « Données » : pourquoi les clips ne sont pas publiés et comment refaire le jeu.

## Le livrable

- [ ] Le dépôt, sans aucune vidéo, avec le protocole de tournage et les annotations
- [ ] Le classifieur entraîné et ses scores sur des clips jamais vus
- [ ] L'API avec sa documentation `/docs` (une capture d'écran dans le README)
- [ ] Les tests, et la coche verte de GitHub Actions
- [ ] Le compte rendu contrôlé
- [ ] Les limites : conditions de tournage, nombre de tireurs, actions non couvertes

## Ce que le projet 5 réutilise

Les tests automatiques, `ruff`, l'habitude de comparer à une référence bête.

## Sources officielles

- MediaPipe, Pose Landmarker en Python : https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker/python
- OpenCV : https://docs.opencv.org/
- scikit-learn, guide de l'utilisateur : https://scikit-learn.org/stable/user_guide.html
- scikit-learn, validation croisée par groupes : https://scikit-learn.org/stable/modules/cross_validation.html
- scikit-learn, pièges courants : https://scikit-learn.org/stable/common_pitfalls.html
- FastAPI : https://fastapi.tiangolo.com/fr/
- FastAPI, tests : https://fastapi.tiangolo.com/tutorial/testing/
- GitHub Actions pour Python : https://docs.github.com/fr/actions/tutorials/build-and-test-code/python
- Ruff : https://docs.astral.sh/ruff/
- Docker, premiers pas : https://docs.docker.com/learn/paths/get-started-with-docker-containers/
- Matplotlib : https://matplotlib.org/stable/
- CNIL, protection des données personnelles : https://www.cnil.fr/fr
