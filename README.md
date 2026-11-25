# Escrime — de la vidéo à une API

> **État : projet en cours.** Les résultats sont remplis au fil des mesures. Une case « à mesurer » n'a pas encore de chiffre.

Une API reçoit une courte vidéo d'un tireur filmé de profil et renvoie l'action reconnue (marche, retraite, fente, garde) avec ses mesures, puis un compte rendu dont chaque chiffre est contrôlé.

## Ce que le projet montre

- Jeu de données filmé et annoté, avec l'accord des personnes
- Estimation de pose, mesures d'angles et de vitesses
- Classifieur évalué sans fuite de données (séparation par clip)
- API FastAPI documentée et testée
- Tests lancés automatiquement par GitHub Actions
- Compte rendu rédigé par un modèle local, chiffres vérifiés par le code

## Résultats mesurés

| Action | Précision | Rappel | Nombre de clips |
|---|---|---|---|
| Marche | à mesurer | | |
| Retraite | à mesurer | | |
| Fente | à mesurer | | |
| Garde | à mesurer | | |

- Date de la mesure : à compléter
- Modèle : à compléter
- Nombre de cas : à compléter
- Commande pour refaire la mesure : à compléter

## Installation

Il faut Python 3.12, [uv](https://docs.astral.sh/uv/) et [Ollama](https://ollama.com/).

```
uv sync
copy .env.example .env
```

Puis écrire dans `.env` le nom du modèle utilisé, téléchargé avec `ollama pull`.

## Utilisation

À compléter : la commande principale et un exemple d'entrée et de sortie.

## Comment ça marche

À compléter : un schéma (Mermaid) et cinq lignes d'explication.

## Limites connues

À compléter : ce qui échoue encore, avec un ou deux exemples.

## Coût

0 €. Le projet tourne en local avec des modèles ouverts. Aucun service payant n'est nécessaire.

## Crédits

- **Code, données et mesures** : Charles Burgy.
- **Cours suivi ([COURS.md](COURS.md)) et structure de départ** : module produit par Claude Code (Anthropic), modèle Claude Fable 5.2, créé le 25/11/2026.
- **Sources officielles** : listées à la fin de [COURS.md](COURS.md).

## Licence

[MIT](LICENSE). Projet d'apprentissage, sans but commercial.
