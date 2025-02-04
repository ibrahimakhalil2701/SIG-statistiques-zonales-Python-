# 📊 Analyse Statistique Zonale avec Python 

Ce projet permet d'effectuer une **analyse zonale** à partir d'un **fichier raster** et vectoriel** contenant des zones géographiques. Il utilise **Python** avec **rasterio, geopandas et rasterstats** pour analyser les valeurs du raster au sein des zones définies.

## 📥 Installation des dépendances

Avant d'exécuter le script, assurez-vous d'avoir installé toutes les bibliothèques nécessaires. Vous pouvez les installer en exécutant la commande suivante :

```bash
!pip install rasterio geopandas numpy pandas rasterstats xlsxwriter
```

## 🚀 Exécution du Script

1. Placez vos fichiers raster (`.tiff`) et shapefile (`.geojson`) dans le répertoire du script.
2. Modifiez les chemins des fichiers dans le script si nécessaire.
3. Exécutez le script Python :

```bash
python script.py
```

## 📜 Code Source

```python

  # Importer les bibliothèques nécessaires
import os
import rasterio
import geopandas as gpd
import numpy as np
import pandas as pd
from rasterio.warp import calculate_default_transform, reproject, Resampling
from rasterstats import zonal_stats
from xlsxwriter import Workbook

# Définition des chemins des fichiers le vectuer ainsi que les deux raster, le programme va faire les stats pour chacun ainsi que la diff des deux 
raster_1991_2020 = "chemin.tiff"
raster_2041_2070 = "chemin.tiff"
shapefile_path = "chemin.geojson"

# Vérification des fichiers s'ils existent 
for file in [raster_1991_2020, raster_2041_2070, shapefile_path]:
    if not os.path.exists(file):
        raise FileNotFoundError(f"❌ Fichier introuvable : {file}")

# Charger le shapefile
gdf = gpd.read_file(shapefile_path)

# Reparation des geometrie si elles sont invalides
# Appliquer la réparation par buffer(0)
gdf["geometry"] = gdf["geometry"].buffer(0)



# Vérifier et reprojeter si nécessaire à adapter selon votre zone d'étude, ici pour Quebec
quebec_lambert_crs = "EPSG:32198"
if gdf.crs != quebec_lambert_crs:
    print(f"⚠️ Reprojection du shapefile en {quebec_lambert_crs}...")
    gdf = gdf.to_crs(quebec_lambert_crs)

# Fonction pour lire un raster et le reprojeter en mémoire si nécessaire
def read_and_reproject_raster(raster_path, target_crs):
    """Charge un raster et le reprojette en mémoire si nécessaire."""
    with rasterio.open(raster_path) as src:
        if src.crs != target_crs:
            print(f"⚠️ Reprojection du raster {raster_path} en {target_crs}...")
            transform, width, height = calculate_default_transform(
                src.crs, target_crs, src.width, src.height, *src.bounds
            )
            profile = src.profile.copy()
            profile.update(crs=target_crs, transform=transform, width=width, height=height)

            raster_data = np.empty((src.count, height, width), dtype=profile["dtype"])
            for i in range(1, src.count + 1):
                reproject(
                    source=rasterio.band(src, i),
                    destination=raster_data[i - 1],
                    src_transform=src.transform,
                    src_crs=src.crs,
                    dst_transform=transform,
                    dst_crs=target_crs,
                    resampling=Resampling.nearest
                )
        else:
            raster_data = src.read()
            transform = src.transform
    return raster_data[0], transform

# Charger et reprojeter les rasters si nécessaire à adapter à vos fichers 
data_1991_2020, transform_1991_2020 = read_and_reproject_raster(raster_1991_2020, quebec_lambert_crs)
data_2041_2070, transform_2041_2070 = read_and_reproject_raster(raster_2041_2070, quebec_lambert_crs)

# Calcul de la différence entre les deux périodes aussi à adapter
raster_difference = data_2041_2070 - data_1991_2020

# Calcul des statistiques zonales pour chaque période aussi à adapter selon vos besoins 
stats_1991_2020 = zonal_stats(
    gdf, data_1991_2020,
    stats=["min", "max", "mean", "std", "median", "percentile_10", "percentile_90"],
    affine=transform_1991_2020
)
stats_2041_2070 = zonal_stats(
    gdf, data_2041_2070,
    stats=["min", "max", "mean", "std", "median", "percentile_10", "percentile_90"],
    affine=transform_2041_2070
)

# Transformer en DataFrame
df_1991_2020 = pd.DataFrame(stats_1991_2020)
df_2041_2070 = pd.DataFrame(stats_2041_2070)

# Ajouter les informations des municipalités, zones Koppen et régions Hydro-Québec, à adapter selon vos besoins
for df in [df_1991_2020, df_2041_2070]:
    df["Type Zone Koppen"] = gdf["intersection_de_classification_hydro_Koppen_geojson_climate_description_fr"]
    df["Region Planification Hydro-Quebec"] = gdf["intersection_de_classification_hydro_Koppen_geojson_region de planification"]
    df["Municipalite"] = gdf["MUS_NM_MUN"]

# Réorganiser les colonnes pour afficher d'abord les informations géographiques
cols = ["Type Zone Koppen", "Region Planification Hydro-Quebec", "Municipalite"] + [col for col in df_1991_2020.columns if col not in ["Municipalite", "Type Zone Koppen", "Region Planification Hydro-Quebec"]]
df_1991_2020 = df_1991_2020[cols]
df_2041_2070 = df_2041_2070[cols]

# Calcul des différences par zone
df_diff = df_2041_2070.copy()
df_diff.iloc[:, 3:] = df_2041_2070.iloc[:, 3:] - df_1991_2020.iloc[:, 3:]

# Export des résultats en Excel
output_excel = "resultats_climat_comparaison.xlsx"
with pd.ExcelWriter(output_excel, engine="xlsxwriter") as writer:
    df_1991_2020.to_excel(writer, sheet_name="Stats_1991_2020", index=False)
    df_2041_2070.to_excel(writer, sheet_name="Stats_2041_2070", index=False)
    df_diff.to_excel(writer, sheet_name="Différences", index=False)

    # Styliser le fichier Excel
    workbook = writer.book
    for sheet in ["Stats_1991_2020", "Stats_2041_2070", "Différences"]:
        worksheet = writer.sheets[sheet]
        worksheet.set_column("A:C", 30)  # Ajuster les colonnes pour Koppen, Hydro, et Municipalité
        worksheet.set_column("D:J", 20)  # Ajuster les colonnes statistiques
        format_header = workbook.add_format({"bold": True, "bg_color": "#DDEBF7", "border": 1})
        for col_num, col_name in enumerate(df_diff.columns):
            worksheet.write(0, col_num, col_name, format_header)
        worksheet.freeze_panes(1, 0)

print(f"✅ Résultat dispo. Résultats enregistrés dans : {output_excel}")

```

## 📂 Structure du Projet

```
/mon_projet
│── script.py                  # Script principal
│── resultats_csv.xlsx       # Résultats générés
│── ficher.tiff        # Raster utilisé
│── vecteur.geojson      # Shapefile utilisé
│── README.md                   # Documentation
```

## 🔥 Fonctionnalités

✔️ Vérification des fichiers avant exécution  
✔️ Reprojection automatique des données spatiales  
✔️ Analyse zonale précise avec rasterstats  
✔️ Génération d'un fichier Excel contenant les résultats  
✔️ Détection automatique des erreurs (ex: superposition manquante)  

## 🛠 Dépendances

- `rasterio`
- `geopandas`
- `numpy`
- `pandas`
- `rasterstats`
- `xlsxwriter`

