# PythonOLAPClient

Python notebook permettant de se connecter à une instance **Power BI Desktop** via ADOMD.NET, d'extraire les métadonnées du modèle tabulaire sémantique et de générer automatiquement un **contrat de données [ODCS v3.1.0](https://github.com/bitol-io/open-data-contract-standard)**.

---

## Table des matières

1. [Prérequis](#prérequis)
2. [Installation des dépendances](#installation-des-dépendances)
3. [Configuration](#configuration)
4. [Utilisation](#utilisation)
5. [Métadonnées extraites](#métadonnées-extraites)
6. [Génération du contrat ODCS](#génération-du-contrat-odcs)
7. [Structure du projet](#structure-du-projet)

---

## Prérequis

| Composant | Version | Notes |
|---|---|---|
| Python | ≥ 3.10 | |
| [.NET](https://dotnet.microsoft.com/en-us/download/dotnet) | 10 | Extraire dans `%LOCALAPPDATA%\Microsoft\dotnet` |
| `pythonnet` | dernière | Chargement du runtime CoreCLR |
| `pyadomd` | dernière | Pilote ADOMD.NET pour Python |
| `pandas` | dernière | Manipulation des données |
| `pyyaml` | dernière | Sérialisation YAML |
| `open-data-contract-standard` | | Modèle ODCS |
| `datacontract-cli` | | Helpers ODCS (`odcs_helper`) |

### Variable d'environnement

```powershell
$Env:DOTNET_ROOT = "$Env:LOCALAPPDATA\Microsoft\dotnet"
```

### Trouver le port Power BI Desktop

Ouvrir le **Gestionnaire des tâches → Détails → AnalysisServices.exe → Moniteur de ressources (TCP)** pour identifier le port d'écoute.

---

## Installation des dépendances

Le notebook contient deux cellules d'initialisation qui téléchargent automatiquement les DLL .NET depuis NuGet.

### 1 — ADOMD.NET Client

```python
PACKAGE_NAME = "Microsoft.AnalysisServices.AdomdClient"
VERSION      = "19.113.2"
```

Télécharge et extrait toutes les DLL `lib/` dans le répertoire courant.

### 2 — Bibliothèques d'identité (MSAL)

```python
LIBRARIES = {
    "MSAL":                  ("Microsoft.Identity.Client",          "4.65.0"),
    "Identity_Abstractions": ("Microsoft.IdentityModel.Abstractions","8.0.0"),
}
```

Priorité accordée aux cibles `netstandard2.0` puis `net6.0` pour la compatibilité cross-platform.

---

## Configuration

Modifier les deux paramètres dans **Cell 6** avant d'exécuter le notebook :

```python
PBIDESKTOP_PORT = "62945"                              # port AnalysisServices.exe
CATALOG_ID      = "66e82ceb-1243-40ef-8b61-fabfa58934ab"  # GUID du dataset ouvert
```

La chaîne de connexion résultante est :

```
Provider=MSOLAP;Data Source=localhost:<PORT>;Initial Catalog=<CATALOG_ID>;
```

---

## Utilisation

Exécuter les cellules dans l'ordre :

| # | Cellule | Action |
|---|---------|--------|
| 0 | Téléchargement ADOMD.NET | Télécharge `Microsoft.AnalysisServices.AdomdClient.dll` |
| 1 | Téléchargement MSAL | Télécharge `Microsoft.Identity.Client.dll` et `Microsoft.IdentityModel.Abstractions.dll` |
| 2 | Note d'installation .NET | Instructions d'installation du runtime .NET |
| 4 | Test de connexion | Connexion rapide et requête DMV `$SYSTEM.TMSCHEMA_COLUMNS` |
| 6 | Paramètres | Définit `PBIDESKTOP_PORT` et `CATALOG_ID` |
| 7 | Imports & helper | Charge `pyadomd`, `pandas`, `yaml` ; définit `run_dmv()` |
| 8 | Extraction | Extrait tables, colonnes, mesures, relations, hiérarchies |
| 9 | Aperçu | Affiche les DataFrames extraits |
| 10 | Mapping de types | Mappe les types tabulaires vers ODCS `logicalType` / `physicalType` |
| 11 | Générateur ODCS | Définit `generate_odcs_contract()` |
| 12 | Génération | Crée le contrat ODCS et affiche un aperçu YAML |
| 13 | Export | Exporte le contrat dans `<nom_catalog>.odcs.yaml` |

---

## Métadonnées extraites

Chaque extraction utilise des requêtes **DMV (Dynamic Management Views)** Analysis Services :

| Objet | DMV | Description |
|---|---|---|
| Informations modèle | `$SYSTEM.DBSCHEMA_CATALOGS` | Nom, description, niveau de compatibilité |
| Tables | `$SYSTEM.TMSCHEMA_TABLES` | Tables non privées (ID, nom, description, visibilité) |
| Colonnes | `$SYSTEM.TMSCHEMA_COLUMNS` | Colonnes de données et calculées (type, DataType, expression DAX) |
| Mesures | `$SYSTEM.TMSCHEMA_MEASURES` | Mesures avec leurs expressions DAX |
| Relations | `$SYSTEM.TMSCHEMA_RELATIONSHIPS` | Cardinalité, filtres croisés, statut actif |
| Hiérarchies | `$SYSTEM.TMSCHEMA_HIERARCHIES` | Hiérarchies utilisateur |

### Types de colonnes (`ColumnType`)

| Code | Type |
|------|------|
| 1 | `data` |
| 2 | `calculated` |
| 4 | `calculatedTableColumn` |

---

## Génération du contrat ODCS

Le contrat généré est conforme à la spécification **[Open Data Contract Standard v3.1.0](https://github.com/bitol-io/open-data-contract-standard)**.

### Paramètres personnalisables (Cell 12)

```python
odcs_contract = generate_odcs_contract(
    contract_name    = None,      # défaut : nom du catalog
    contract_version = '1.0.0',
    status           = 'draft',   # 'draft' | 'active' | 'deprecated'
    domain           = '',
    owner            = '',
    description      = '',
)
```

### Structure du contrat généré

```yaml
# Generated by PythonOLAPClient — <timestamp UTC>
# Standard : Open Data Contract Standard (ODCS) v3.1.0
# Source   : Power BI Desktop | port <PORT> | catalog <CATALOG_ID>

id: <uuid>
name: <catalog_name>
version: 1.0.0
status: draft
servers:
  - name: Power BI Desktop
    type: custom
    host: localhost:<PORT>
    database: <catalog_id>
schema:
  - id: tbl_<id>
    name: <TableName>
    physicalType: table
    properties:
      - id: col_<id>
        name: <ColumnName>
        logicalType: string       # mappé depuis le DataType tabulaire
        physicalType: WCHAR
        tags: [data]
        relationships:
          - to: OtherTable.OtherColumn
            type: foreignKey
      - id: meas_<id>
        name: <MeasureName>
        logicalType: number
        physicalType: measure
        transformLogic: "<expression DAX>"
        tags: [measure]
```

### Mapping des types tabulaires → ODCS

| DataType | logicalType | physicalType |
|----------|-------------|--------------|
| 2 | integer | SHORT |
| 7 | date | DATE |
| 8 | string | BSTR |
| 11 | boolean | BOOL |
| 14 | number | DECIMAL |
| 130 | string | WCHAR |
| 133 | date | DATETIME |
| 135 | timestamp | TIMESTAMP |

Le fichier de sortie est nommé `<nom_catalog>.odcs.yaml` et placé dans le répertoire courant.

---

## Structure du projet

```
PowerBI_PythonClient/
├── PythonOLAPClient.ipynb   # Notebook principal
├── <catalog>.odcs.yaml      # Contrat ODCS généré (après exécution)
└── *.dll                    # DLL .NET téléchargées depuis NuGet
```
