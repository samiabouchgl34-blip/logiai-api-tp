# Compte rendu : TP API LogiAI

**Noms :** Bouchgl, Eddhibi  **Groupe :** 4

## 1. Environnement et premiers tests
- Sortie de `python -m pytest` au départ (collez-la) :

```
-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================================================== short test summary info =========================================================
FAILED tests/test_api.py::test_predict_valide_renvoie_200 - assert 422 == 200
FAILED tests/test_api.py::test_predict_forme_de_la_reponse - AssertionError: assert {'detail'} == {'alerte', 'm...ard', 'seuil'}
FAILED tests/test_api.py::test_predict_probabilite_entre_0_et_1 - KeyError: 'probabilite_retard'
FAILED tests/test_api.py::test_predict_trajet_risque_plus_eleve_que_trajet_facile - KeyError: 'probabilite_retard'
FAILED tests/test_api.py::test_modele_absent_renvoie_503 - assert 422 == 503
================================================ 5 failed, 8 passed, 1 warning in 12.52s =================================================
```

- Pourquoi certains tests qui attendent un 422 passent-ils déjà, alors que rien n'est écrit ?

  Le schéma de départ est presque vide et refuse tout champ inconnu (`extra="forbid"`).
  Toute requête reçoit donc un 422, même une livraison valide. Les tests de rejet voient
  bien le 422 qu'ils attendent, mais pour la mauvaise raison : ce n'est pas une validation
  qui les a refusés, c'est un schéma cassé qui refuse tout.
  Le test de succès `test_predict_valide_renvoie_200` les démasque : il attend un 200 et
  reçoit un 422.
  Ce que ça révèle : un test vert ne prouve rien tant qu'on ne sait pas pourquoi il passe.
  Un test de rejet (422) doit toujours être accompagné d'un test d'acceptation (200),
  sinon un code qui refuse tout passe inaperçu.

- Résultats de `python -m logiai.train` : accuracy baseline 0,726 (toujours « à l'heure »),
  accuracy du modèle 0,774, rappel 0,464 (environ 46 retards détectés sur 100).

## 2. Mon schéma
- Mes bornes et leur justification physique (une ligne par variable) :

| Variable | Min | Max | Pourquoi |
|---|---|---|---|
| distance_km | 1 | 1000 | un trajet fait au moins 1 km ; au-delà de 1000 km, hors périmètre de la société |
| poids_kg | 0,1 | 40000 | 100 g pour un petit colis ; 40 t, charge maximale d'un camion |
| heure_depart | 0 | 23 | heure entière d'une journée, comme dans le contrat |
| pluie_mm | 0 | 300 | pas de pluie négative ; 300 mm est déjà un événement extrême |
| trafic_index | 0 | 10 | défini par le contrat : 0 fluide, 10 saturé |
| nb_arrets | 0 | 50 | entier ; 50 arrêts est déjà une tournée très chargée |

## 3. Ma fonction predict()

```python
@app.post("/v1/predict", response_model=PredictionOut)
def predict(livraison: Livraison):
    predictor = state.get("predictor")
    if predictor is None:
        raise HTTPException(
            status_code=503,
            detail="Modèle non chargé : lancez python -m logiai.train puis relancez le serveur",
        )
    return PredictionOut(**predictor.predict(livraison.model_dump()))
```

## 4. Quatre requêtes
| Requête | Code obtenu | Pourquoi |
|---|---|---|
| livraison normale | 200 | entrée valide, le modèle répond avec la probabilité |
| distance_km = -5 | 422 | refusée par le schéma (`ge=1`) avant d'appeler `predict()` |
| type_vehicule = "avion" | 422 | pas dans `Literal["moto", "fourgon", "camion"]` |
| modèle absent | /health 200 (model_loaded: false), /v1/predict 503 | l'API tourne et reçoit une requête valide, mais ne peut pas prédire sans modèle |

## 5. Questions de compréhension
1. Lire et désérialiser le fichier du modèle à chaque requête est lent et gaspille de la
   mémoire dès que plusieurs clients appellent. Chargé une seule fois au démarrage
   (`lifespan`), le modèle reste en mémoire et chaque requête répond vite.
2. C'est le schéma Pydantic qui écrit ce contrôle, dans `logiai/api/schemas.py`, avec
   `Field(ge=0, le=300)` sur le champ `pluie_mm` (ma borne est 300, pas 100). FastAPI
   l'applique avant d'appeler `predict()` : une pluie de 500 mm reçoit un 422.
3. 422 : la requête du client est invalide, l'erreur vient de lui (4xx). 503 : la requête
   est valide mais le service est temporairement incapable de répondre (5xx), ici parce
   que le modèle n'est pas chargé.
4. `/v1` est la version de l'API : le contrat peut évoluer (`/v2`) sans casser les clients
   qui utilisent encore `/v1`.
5. Une distance décimale comme 12,5 serait refusée en 422, alors que c'est une valeur
   normale. Les clients qui envoient des décimales seraient bloqués à tort.

## 6. Décision : modèle absent
Répondre 503 ou refuser de démarrer ? Qu'est-ce que chaque option fait perdre ?

**Mon choix : l'option A**, démarrer quand même et répondre 503 aux demandes de prédiction.

- Si l'API refuse de démarrer (option B), on perd `/health` : la personne qui supervise
  ne voit qu'un service éteint et doit deviner pourquoi (modèle manquant, port occupé,
  bug dans le code ?). Les clients n'ont pas non plus de message clair, juste une
  connexion refusée.
- Si l'API démarre sans modèle (option A), on perd la visibilité immédiate du problème :
  le service paraît en marche alors qu'il ne peut rien prédire.

Je choisis A parce qu'elle laisse `/health` répondre avec `model_loaded: false`, donc la
panne est identifiable tout de suite. Le code 503 dit aussi aux clients que le service est
temporairement indisponible, sans le confondre avec une faute de leur part (4xx) ni avec un
bug (500). Le risque de A, un service « up » mais inutile, se corrige en faisant surveiller
le champ `model_loaded`.

**Mes deux tests** (dans `tests/test_api.py`) :

```python
def test_heure_24_refusee():
    with TestClient(app) as client:
        r = client.post("/v1/predict", json={**VALIDE, "heure_depart": 24})
        assert r.status_code == 422
        r = client.post("/v1/predict", json={**VALIDE, "heure_depart": 23})
        assert r.status_code == 200


def test_alerte_coherente_avec_le_seuil():
    with TestClient(app) as client:
        difficile = client.post("/v1/predict", json=DIFFICILE).json()
        facile = client.post("/v1/predict", json=FACILE).json()
        assert difficile["alerte"] is True
        assert facile["alerte"] is False
```

Le premier vérifie que 24 est refusée (et que 23 passe, pour éviter un faux vert). Le second
vérifie que le trajet très difficile déclenche l'alerte et pas le trajet très facile.
Contrôle : avec `le=24` dans le schéma, `test_heure_24_refusee` est devenu rouge ; avec
`le=23`, il est redevenu vert.

## 7. Lien de la pull request
