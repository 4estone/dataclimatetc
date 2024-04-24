# ForEverPollution méthodologie

![62d7f4694011262d9176ff0767ec0795.png](../_resources/62d7f4694011262d9176ff0767ec0795.png)

### Liens et ressources originaux:

Le projet : https://foreverpollution.eu/  
Article du monde présentant les résultats du projet : https://www.lemonde.fr/les-decodeurs/article/2023/02/23/revelations-sur-la-contamination-massive-de-l-europe-par-les-pfas-ces-polluants-eternels_6162940_4355770.html

### Les données

Ici: https://lucmartinon.gitlab.io/ffp-data/  
Disponibles au format [CSV](https://fr.wikipedia.org/wiki/Comma-separated_values) et [parquet](https://fr.wikipedia.org/wiki/Apache_Parquet). Différence significative de taille: 375 Mo pour le CSV et 25 Mo pour le parquet.

On est parti pour le parquet!  
Un simple glissé/déposé dans QGIS permet une visualisation directe des données alphanumériques.  
![cc7a52908d4e2794aa8c5394f075ed57.png](../_resources/cc7a52908d4e2794aa8c5394f075ed57.png)

#### Première analyse

- 337441 enregistrements
- 133281 enregistrement avec une valeur totale de PFAS > 0
- pas de clé primaire
- plusieurs enregistrements pour un même point de mesure
- 2 champs contenant des données au format json

#### Traitement des données

- Créer une clé primaire
- Regrouper les points de mesure sur tous leurs champs communs.
- Récupérer la valeur maximum mesurée par point de mesure. 
- Créer la localisation géographique.

On va faire tout çà en une requête SQL dans le gestionnaire de base de données de QGIS:

```
SELECT 
    CAST(full.lon AS TEXT) || CAST(full.lat as TEXT) || COALESCE(full.name, '') AS id,
    full.category,
    full.lat,
    full.lon,
    full.name,
    full.city,
    full.country,
    full.type,
    full.sector,
    full.source_type,
    full.source_text,
    full.source_url,
    full.dataset_id,
    full.dataset_name,
    max(full.pfas_sum) as pfas_sum,
    full.matrix, 
    MakePoint(full.lon, full.lat, 4326) AS geom
FROM full 
WHERE full.pfas_sum > 0 AND full.lat is not null AND full.lon is not null
GROUP BY 
    id,
    full.category,
    full.lat,
    full.lon,
    full.name,
    full.city,
    full.country,
    full.type,
    full.sector,
    full.source_type,
    full.source_text,
    full.source_url,
    full.dataset_id,
    full.dataset_name,
    full.matrix
```

On charge le résultat en tant que couche virtuelle, puis enregistrer ce résultat dans un geopackage.

![276094fbdd8e276ab93d11559ede0628.png](../_resources/276094fbdd8e276ab93d11559ede0628.png)
#### Représentation cartographique
La valeur à représenter est le maximum de PFAS rencontré pour un point de mesure.La plage de valeurs s'étire de quelques centième de nanogramme à 80000000 (!). Le choix d'une représentation par échelle logarithmique s'impose donc.
![7feb423f7bced8dbc836204f09c9be08.png](../_resources/7feb423f7bced8dbc836204f09c9be08.png)

Et voilà! Une carte dynamique à votre main vous permettant d'explorer tranquillement les données.

![45732e60b92fe57020847956a508943f.png](../_resources/45732e60b92fe57020847956a508943f.png)

Mais ce n'est pas fini. Il faut rendre accessible toutes les valeurs mesurées sur un même point.
On repart dans le gestionnaire de base de données pour une autre requête SQL, objectifs:
- Créer une clé primaire identique à celle créée sur la première requête
- récupérer les valeurs de PFAS 
```
SELECT 
    CAST(full.lon AS TEXT) || CAST(full.lat as TEXT) || COALESCE(full.name, '') AS id,
    cast(full.year as integer) as year, 
    full.date, 
    full.pfas_sum,
    full.pfas_values,
    full.details
FROM full 
where full.pfas_sum > 0 and full.lat is not null and full.lon is not null
GROUP BY 
    id,
	full.year, 
    full.date, 
    full.pfas_values,
    full.details
```

On charge le résultat en tant que couche virtuelle, puis on enregistre ce résultat dans le geopackage existant.

![6247f2b343a4ef327a766aab1a1e17d6.png](../_resources/6247f2b343a4ef327a766aab1a1e17d6.png)

#### Mise en place de la relation entre les tables *pfas_value* et *PFAS_sites*

![e2e17e63784f0f9e25eddd6516908cd4.png](../_resources/e2e17e63784f0f9e25eddd6516908cd4.png)

Les valeurs individuelles sont maintenant accessibles dans le formulaire d'interrogation individuelle des objets
![b3972d2aea71241571b886f54e8491fb.png](../_resources/b3972d2aea71241571b886f54e8491fb.png)
