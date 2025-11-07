# 📚 Chapitre 27 : Annexes - Codes OBIS du compteur E450

## 🔍 Répertoire complet des codes OBIS

Le compteur Landis+Gyr E450 expose plus de **100 codes OBIS** différents, chacun correspondant à une mesure spécifique ou à une information système. Cette annexe présente un catalogue exhaustif organisé par catégories.

## 📊 Énergies cumulées (Wh/kWh/MWh)

### Import actif (consommation)

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **1.8.0** | Énergie active importée totale | kWh | Index principal de facturation |
| **1.8.1** | Énergie active importée tarif 1 | kWh | Heures pleines/creuses |
| **1.8.2** | Énergie active importée tarif 2 | kWh | Tarif secondaire |
| **1.8.3** | Énergie active importée tarif 3 | kWh | Tarif tertiaire |
| **1.8.4** | Énergie active importée tarif 4 | kWh | Tarif quaternaire |
| **1.8.5** | Énergie active importée tarif 5 | kWh | Tarif supplémentaire |
| **1.8.6** | Énergie active importée tarif 6 | kWh | Tarif étendu |
| **1.8.7** | Énergie active importée tarif 7 | kWh | Tarif spécial |
| **1.8.8** | Énergie active importée tarif 8 | kWh | Tarif maximum |

### Export actif (production/injection)

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **2.8.0** | Énergie active exportée totale | kWh | Injection réseau |
| **2.8.1** | Énergie active exportée tarif 1 | kWh | Injection HP |
| **2.8.2** | Énergie active exportée tarif 2 | kWh | Injection HC |
| **2.8.3** | Énergie active exportée tarif 3 | kWh | Injection tarif 3 |
| **2.8.4** | Énergie active exportée tarif 4 | kWh | Injection tarif 4 |

### Énergies réactives

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **3.8.0** | Énergie réactive importée totale | kVArh | Réactive inductive |
| **4.8.0** | Énergie réactive exportée totale | kVArh | Réactive capacitive |
| **5.8.0** | Énergie réactive quadrante 1 | kVArh | Import actif/réactif |
| **6.8.0** | Énergie réactive quadrante 2 | kVArh | Export actif/réactif |
| **7.8.0** | Énergie réactive quadrante 3 | kVArh | Import réactif/actif |
| **8.8.0** | Énergie réactive quadrante 4 | kVArh | Export réactif/actif |

### Énergies apparentes

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **9.8.0** | Énergie apparente totale | kVAh | S = √(P² + Q²) |
| **9.8.1** | Énergie apparente tarif 1 | kVAh | Apparente HP |
| **9.8.2** | Énergie apparente tarif 2 | kVAh | Apparente HC |

## ⚡ Puissances instantanées (W/VA/VAr)

### Puissances actives

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **16.7.0** | Puissance active totale | W | Puissance consommée |
| **16.7.1** | Puissance active phase 1 | W | Phase L1 |
| **16.7.2** | Puissance active phase 2 | W | Phase L2 |
| **16.7.3** | Puissance active phase 3 | W | Phase L3 |
| **16.7.4** | Puissance active neutre | W | Fil neutre |

### Puissances réactives

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **36.7.0** | Puissance réactive totale | VAr | Puissance réactive |
| **36.7.1** | Puissance réactive phase 1 | VAr | Réactive L1 |
| **36.7.2** | Puissance réactive phase 2 | VAr | Réactive L2 |
| **36.7.3** | Puissance réactive phase 3 | VAr | Réactive L3 |

### Puissances apparentes

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **56.7.0** | Puissance apparente totale | VA | Puissance apparente |
| **56.7.1** | Puissance apparente phase 1 | VA | Apparente L1 |
| **56.7.2** | Puissance apparente phase 2 | VA | Apparente L2 |
| **56.7.3** | Puissance apparente phase 3 | VA | Apparente L3 |

## 🔌 Tensions (V)

### Tensions efficaces

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **32.7.0** | Tension phase 1 | V | Tension L1-N |
| **32.7.1** | Tension phase 1 (détaillée) | V | L1 avec précision |
| **52.7.0** | Tension phase 2 | V | Tension L2-N |
| **52.7.1** | Tension phase 2 (détaillée) | V | L2 avec précision |
| **72.7.0** | Tension phase 3 | V | Tension L3-N |
| **72.7.1** | Tension phase 3 (détaillée) | V | L3 avec précision |
| **13.7.0** | Tension moyenne | V | Moyenne 3 phases |

### Tensions entre phases

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **32.7.2** | Tension L1-L2 | V | Phase à phase |
| **52.7.2** | Tension L2-L3 | V | Phase à phase |
| **72.7.2** | Tension L3-L1 | V | Phase à phase |

## 🔌 Courants (A)

### Courants efficaces

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **31.7.0** | Courant phase 1 | A | Intensité L1 |
| **31.7.1** | Courant phase 1 (détaillé) | A | L1 haute précision |
| **51.7.0** | Courant phase 2 | A | Intensité L2 |
| **51.7.1** | Courant phase 2 (détaillé) | A | L2 haute précision |
| **71.7.0** | Courant phase 3 | A | Intensité L3 |
| **71.7.1** | Courant phase 3 (détaillé) | A | L3 haute précision |
| **91.7.0** | Courant neutre | A | Intensité fil neutre |

### Courants de démarrage

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **31.7.2** | Courant de démarrage L1 | A | Seuil démarrage |
| **51.7.2** | Courant de démarrage L2 | A | Seuil démarrage |
| **71.7.2** | Courant de démarrage L3 | A | Seuil démarrage |

## 🌡️ Qualité du réseau

### Fréquence

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **14.7.0** | Fréquence réseau | Hz | Fréquence électrique |
| **14.7.1** | Fréquence (détaillée) | Hz | Haute précision |

### Facteur de puissance

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **13.21.0** | Facteur de puissance total | - | cos φ global |
| **13.21.1** | Facteur de puissance L1 | - | cos φ phase 1 |
| **13.21.2** | Facteur de puissance L2 | - | cos φ phase 2 |
| **13.21.3** | Facteur de puissance L3 | - | cos φ phase 3 |

### Distorsion harmonique (THD)

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **7.7.0** | THD tension totale | % | Distorsion harmonique |
| **7.7.1** | THD tension L1 | % | Distorsion L1 |
| **7.7.2** | THD tension L2 | % | Distorsion L2 |
| **7.7.3** | THD tension L3 | % | Distorsion L3 |
| **8.7.0** | THD courant total | % | Distorsion courant |
| **8.7.1** | THD courant L1 | % | Distorsion I L1 |
| **8.7.2** | THD courant L2 | % | Distorsion I L2 |
| **8.7.3** | THD courant L3 | % | Distorsion I L3 |

### Puissance maximale

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **16.7.4** | Puissance maximale | W | Pic historique |
| **16.7.5** | Puissance maximale journalière | W | Pic du jour |
| **16.7.6** | Puissance maximale mensuelle | W | Pic du mois |

## ⏰ Informations temporelles

### Date et heure

| Code OBIS | Description | Format | Remarques |
|-----------|-------------|--------|-----------|
| **0.9.1** | Date compteur | YYMMDD | Date système |
| **0.9.2** | Heure compteur | HHMMSS | Heure système |
| **0.9.13** | Timestamp complet | YYMMDDHHMMSS | Date/heure complète |

### Fuseau horaire

| Code OBIS | Description | Format | Remarques |
|-----------|-------------|--------|-----------|
| **0.9.15** | Décalage horaire | +HHMM/-HHMM | UTC offset |

## 📋 Profils de charge (historique)

### Profils horaires

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **15.7.0** | Profil de charge actif | W | Historique puissance |
| **15.7.1** | Profil de charge réactif | VAr | Historique réactive |
| **15.7.2** | Profil de charge apparent | VA | Historique apparente |

### Intervalle de mesure

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **15.7.3** | Profil 15 minutes | W | Intervalle court |
| **15.7.4** | Profil 30 minutes | W | Intervalle moyen |
| **15.7.5** | Profil horaire | W | Intervalle horaire |
| **15.7.6** | Profil journalier | Wh | Cumul journalier |

## 🔧 Informations système

### Numéros de série et identification

| Code OBIS | Description | Exemple | Remarques |
|-----------|-------------|---------|-----------|
| **0.0.0** | Numéro de série | E450001234 | Identifiant unique |
| **0.0.1** | Version logicielle | 3.4.5 | Firmware version |
| **0.0.2** | Version hardware | 1.2 | Matériel version |
| **96.1.0** | Fabricant | LGZ | Code Landis+Gyr |
| **96.1.1** | Modèle | E450 | Type de compteur |

### État du compteur

| Code OBIS | Description | Valeurs | Remarques |
|-----------|-------------|---------|-----------|
| **96.5.0** | Statut général | OK/ERROR | État global |
| **96.5.1** | Statut mémoire | OK/ERROR | Mémoire interne |
| **96.5.2** | Statut horloge | OK/ERROR | RTC état |
| **96.5.3** | Statut communication | OK/ERROR | Interfaces |

### Compteurs d'événements

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **97.97.0** | Nombre d'événements | - | Total événements |
| **97.97.1** | Événements programmation | - | Changements config |
| **97.97.2** | Événements puissance | - | Pics/chutes |
| **97.97.3** | Événements tension | - | Anomalies tension |
| **97.97.4** | Événements courant | - | Anomalies courant |

## 🔐 Sécurité et authentification

### Niveaux d'accès

| Code OBIS | Description | Valeurs | Remarques |
|-----------|-------------|---------|-----------|
| **0.1.0** | Niveau d'accès actuel | 1-4 | Niveau authentifié |
| **0.1.1** | Tentatives d'accès | - | Compteur tentatives |
| **0.1.2** | Dernière authentification | timestamp | Dernière connexion |

### Codes d'accès

| Code OBIS | Description | Format | Remarques |
|-----------|-------------|--------|-----------|
| **0.2.0** | Code accès lecteur | string | Lecture seule |
| **0.2.1** | Code accès opérateur | string | Paramètres |
| **0.2.2** | Code accès configurateur | string | Configuration |

## 📊 Données de diagnostic

### Tests automatiques

| Code OBIS | Description | Résultat | Remarques |
|-----------|-------------|----------|-----------|
| **97.98.0** | Test mémoire | OK/ERROR | RAM/EEPROM |
| **97.98.1** | Test horloge | OK/ERROR | RTC précision |
| **97.98.2** | Test mesure | OK/ERROR | Circuits mesure |
| **97.98.3** | Test communication | OK/ERROR | Interfaces |

### Métrologie

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **97.99.0** | Classe de précision | - | Classe B |
| **97.99.1** | Erreur maximale | % | Erreur tolérée |
| **97.99.2** | Plage de mesure | A/V | Limites mesure |

## 🔄 Paramètres configurables

### Configuration tarifaire

| Code OBIS | Description | Valeurs | Remarques |
|-----------|-------------|---------|-----------|
| **0.4.0** | Nombre de tarifs | 1-8 | Tarifs actifs |
| **0.4.1** | Tarif actif | 1-8 | Tarif courant |
| **0.4.2** | Programmation tarifaire | schedule | Calendrier tarifs |

### Seuils et limites

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **0.5.0** | Seuil puissance max | W | Limite alerte |
| **0.5.1** | Seuil tension min | V | Tension basse |
| **0.5.2** | Seuil tension max | V | Tension haute |
| **0.5.3** | Seuil courant max | A | Surcharge |

## 📡 Communication et réseau

### Adresses et identifiants

| Code OBIS | Description | Format | Remarques |
|-----------|-------------|--------|-----------|
| **96.1.2** | Adresse physique | hex | Adresse matériel |
| **96.1.3** | Adresse réseau | IP/MAC | Configuration réseau |
| **96.1.4** | Adresse secondaire M-Bus | 001-250 | Bus M-Bus |

### Statistiques communication

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **97.96.0** | Trames reçues | - | Total reçues |
| **97.96.1** | Trames envoyées | - | Total envoyées |
| **97.96.2** | Erreurs communication | - | Nombre erreurs |
| **97.96.3** | Taux d'erreur | % | Qualité liaison |

## 🏭 Données spécifiques fabricant

### Landis+Gyr spécifiques

| Code OBIS | Description | Valeurs | Remarques |
|-----------|-------------|---------|-----------|
| **96.99.0** | Code fabricant | LGZ | Landis+Gyr |
| **96.99.1** | Pays d'origine | CH | Suisse |
| **96.99.2** | Année fabrication | YYYY | Date production |
| **96.99.3** | Numéro de lot | string | Lot production |

### Données de maintenance

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **97.100.0** | Heures fonctionnement | h | Durée vie |
| **97.100.1** | Cycles marche/arrêt | - | Nombre redémarrages |
| **97.100.2** | Température interne max | °C | Chaleur interne |
| **97.100.3** | Nombre de calibrations | - | Historique calibration |

## 📋 Guide d'utilisation pratique

### Codes essentiels pour monitoring de base

```python
CODES_ESSENTIELS = [
    '1.8.0',   # Énergie totale importée
    '2.8.0',   # Énergie totale exportée
    '16.7.0',  # Puissance active actuelle
    '32.7.0',  # Tension L1
    '52.7.0',  # Tension L2
    '72.7.0',  # Tension L3
    '31.7.0',  # Courant L1
    '51.7.0',  # Courant L2
    '71.7.0',  # Courant L3
    '14.7.0',  # Fréquence
    '13.21.0', # Facteur de puissance
]
```

### Codes pour analyse qualité réseau

```python
CODES_QUALITE = [
    '7.7.0',   # THD tension totale
    '8.7.0',   # THD courant total
    '13.7.0',  # Tension moyenne
    '97.98.0', # Test mémoire
    '97.98.2', # Test circuits mesure
]
```

### Codes pour diagnostic avancé

```python
CODES_DIAGNOSTIC = [
    '0.0.0',   # Numéro de série
    '96.5.0',  # Statut général
    '97.97.0', # Nombre d'événements
    '97.100.0', # Heures fonctionnement
    '97.100.1', # Cycles marche/arrêt
]
```

## 🔍 Recherche et filtrage

### Recherche par catégorie

```python
def filtrer_codes_par_categorie(codes_obis, categorie):
    """Filtre les codes OBIS par catégorie"""
    categories = {
        'energie': lambda c: c.split('.')[1] == '8',
        'puissance': lambda c: c.split('.')[1] == '7' and c.split('.')[0] in ['16', '36', '56'],
        'tension': lambda c: c.split('.')[0] in ['32', '52', '72'],
        'courant': lambda c: c.split('.')[0] in ['31', '51', '71', '91'],
        'qualite': lambda c: c.split('.')[0] in ['7', '8', '13', '14'],
        'systeme': lambda c: c.split('.')[0] in ['0', '96', '97'],
    }

    if categorie in categories:
        return [c for c in codes_obis if categories[categorie](c)]
    return codes_obis
```

### Recherche par mot-clé

```python
def rechercher_codes_par_mot(codes_obis, descriptions, mot_cle):
    """Recherche de codes par mot-clé dans la description"""
    resultats = []
    mot_cle_lower = mot_cle.lower()

    for code, desc in descriptions.items():
        if mot_cle_lower in desc.lower():
            resultats.append((code, desc))

    return resultats
```

## 📊 Statistiques des codes OBIS

### Répartition par type

- **Énergies cumulées** : 25 codes (25%)
- **Puissances instantanées** : 15 codes (15%)
- **Tensions** : 10 codes (10%)
- **Courants** : 9 codes (9%)
- **Qualité réseau** : 12 codes (12%)
- **Système & diagnostic** : 30 codes (30%)
- **Communication** : 8 codes (8%)
- **Fabricant** : 6 codes (6%)

### Codes les plus utilisés

1. **1.8.0** - Énergie active importée totale
2. **16.7.0** - Puissance active totale
3. **32.7.0** - Tension phase 1
4. **31.7.0** - Courant phase 1
5. **14.7.0** - Fréquence réseau
6. **13.21.0** - Facteur de puissance
7. **0.0.0** - Numéro de série
8. **96.5.0** - Statut compteur
9. **97.97.0** - Événements
10. **15.7.0** - Profil de charge

> **💡 À retenir** : Les 100+ codes OBIS du E450 forment un système complet de monitoring énergétique, permettant à la fois le suivi de base et l'analyse avancée de la qualité du réseau.

> **⚠️ Astuce** : Commencez par maîtriser les 10 codes essentiels avant d'explorer les codes spécialisés pour le diagnostic et l'analyse fine.

Cette annexe constitue une référence complète pour exploiter pleinement les capacités du compteur Landis+Gyr E450 !

---

**Navigation**
- [Chapitre précédent : Prévision énergétique avec Prophet](../Partie_V_Analyse_Visualisation/Chapitre_23_Preuve_Energetique_Prophet.md)
- [Chapitre suivant : Atelier industriel](Chapitre_25_Atelier_Industriel.md)
- [Retour à la table des matières](../../README.md)
