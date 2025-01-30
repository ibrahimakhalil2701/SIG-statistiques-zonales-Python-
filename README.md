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
import os
import rasterio
import geopandas as gpd
import numpy as np
import pandas as pd
from rasterio.warp import calculate_default_transform, reproject, Resampling
from rasterstats import zonal_stats

# Définition des chemins des fichiers
raster_path = "ficher.tiff"
shapefile_path = "verteur.geojson"

# Définir le CRS NAD83 / Québec Lambert (EPSG:32198)  selon votre zone 
quebec_lambert_crs = "EPSG:32198"

# Vérification des fichiers
if not os.path.exists(raster_path):
    raise FileNotFoundError(f"\u274c Le fichier raster est introuvable : {raster_path}")
if not os.path.exists(shapefile_path):
    raise FileNotFoundError(f"\u274c Le fichier GeoJSON est introuvable : {shapefile_path}")

# Charger le vecteur
gdf = gpd.read_file(shapefile_path)

# Vérifier si le vecteur est vide
if gdf.empty:
    raise ValueError("\u274c Le shapefile est vide après lecture.")

# Reprojeter le vecteur si nécessaire vers les crs de votre zone
if gdf.crs != quebec_lambert_crs:
    print(f"⚠️ Reprojection du shapefile de {gdf.crs} vers {quebec_lambert_crs}...")
    gdf = gdf.to_crs(quebec_lambert_crs)

# Reprojection du raster en mémoire
def reproject_raster_to_lambert(src_path, target_crs):
    with rasterio.open(src_path) as src:
        transform, width, height = calculate_default_transform(
            src.crs, target_crs, src.width, src.height, *src.bounds
        )
        profile = src.profile
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
    return raster_data, transform, profile

# Reprojeter le raster
print("🔄 Reprojection du raster en cours...")
raster_data, transform, profile = reproject_raster_to_lambert(raster_path, quebec_lambert_crs)
print("✅ Reprojection du raster terminée.")

# Vérification de la superposition raster/vecteur à modifier ou à enlever selon votre cas
gdf_bounds = gdf.total_bounds  # xmin, ymin, xmax, ymax

raster_bounds = (
    transform[2],  # Min X
    transform[5] + transform[4] * raster_data.shape[1],  # Min Y
    transform[2] + transform[0] * raster_data.shape[2],  # Max X
    transform[5],  # Max Y
)

buffer_tolerance = 1000  # Tolérance en mètres
if not (
    gdf_bounds[0] - buffer_tolerance < raster_bounds[2] and
    gdf_bounds[2] + buffer_tolerance > raster_bounds[0] and
    gdf_bounds[1] - buffer_tolerance < raster_bounds[3] and
    gdf_bounds[3] + buffer_tolerance > raster_bounds[1]
):
    raise ValueError("\u274c Le raster et le shapefile ne se superposent pas (même avec tolérance).")
else:
    print("✅ Superposition correcte.")

# Analyse zonale
try:
    print("🔄 Lancement de l'analyse zonale...")
    stats = zonal_stats(
        gdf, raster_data[0],  
        stats=["min", "max", "mean", "std", "median"],
        nodata=None,
        affine=transform,  
        geojson_out=True
    )
    print("✅ Analyse zonale réussie.")
except Exception as e:
    raise RuntimeError(f"\u274c Erreur lors de l'analyse zonale : {e}")

# Export des résultats en Excel
output_excel = "resultats_climat.xlsx"
with pd.ExcelWriter(output_excel, engine="xlsxwriter") as writer:
    df_stats.to_excel(writer, sheet_name="Statistiques", index=False)
    workbook = writer.book
    worksheet = writer.sheets["Statistiques"]
    worksheet.set_column("A:A", 30)
    worksheet.set_column("B:F", 20)
    format_header = workbook.add_format({"bold": True, "bg_color": "#DDEBF7", "border": 1})
    for col_num, col_name in enumerate(df_stats.columns):
        worksheet.write(0, col_num, col_name, format_header)
    worksheet.freeze_panes(1, 0)

print(f"✅ Fichier Excel généré : {output_excel}")
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

