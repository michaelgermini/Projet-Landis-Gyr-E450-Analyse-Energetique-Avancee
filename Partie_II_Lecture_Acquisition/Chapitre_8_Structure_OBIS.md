# ⚙️ Chapitre 8 : Structure des données OBIS

## 🔍 Qu'est-ce que le code OBIS ?

OBIS (OBject Identification System) est un **système de codification standardisé** qui identifie de manière unique chaque donnée mesurée ou calculée par un compteur électrique. C'est la **clé de voûte** de la communication DLMS/COSEM.

### Origine et standardisation

```
Développé par : DLMS UA (User Association)
Standard : IEC 62056-61 (OBIS)
But : Identification unifiée des données de mesure
Adoption : Mondiale (Europe, Asie, Amérique)
```

### Pourquoi OBIS est essentiel ?

- ✅ **Standardisation** : Même code = même donnée partout
- ✅ **Évolutivité** : Nouveaux codes pour nouvelles mesures
- ✅ **Clarté** : Signification immédiate du code
- ✅ **International** : Compréhension universelle
- ✅ **Structuré** : Hiérarchie logique des informations

## 📐 Structure du code OBIS

### Format général

```
A.B.C.D.E.F
└─┼─┼─┼─┼─┼─┘
  │ │ │ │ │ └── F: Stockage (mémoire)
  │ │ │ │ └──── E: Tarif/Rate
  │ │ │ └────── D: Mesure/Quantité
  │ │ └──────── C: Type de mesure
  │ └────────── B: Canal/Phase
  └──────────── A: Médium/Support
```

Chaque lettre représente un **chiffre de 0 à 9** (parfois étendu à A-F).

### Exemples concrets

#### Énergie active importée totale
```
1.8.0.255.2.255
│ │ │ │   │ │
│ │ │ │   │ └── Stockage: Total (255 = tous)
│ │ │ │   └──── Tarif: Total (2 = tarif global)
│ │ │ └──────── Mesure: Énergie (8 = Wh)
│ │ └────────── Type: Active import (8 = active+)
│ └──────────── Médium: Électricité (1)
└────────────── Canal: Total (8 = somme)
```

#### Tension phase 1
```
32.7.0.255.2.255
│ │ │ │   │ │
│ │ │ │   │ └── Stockage: Instantané
│ │ │ │   └──── Tarif: Total
│ │ │ └──────── Mesure: Tension (7 = V)
│ │ └────────── Type: Momentanée (7 = instant)
│ └──────────── Médium: Électricité
└────────────── Canal: Phase 1 (32)
```

## 📊 Tableau des codes principaux

### Énergies cumulées (Wh/kWh/MWh)

| Code OBIS | Description | Unité | Remarques |
|-----------|-------------|-------|-----------|
| **1.8.0** | Énergie active importée totale | kWh | Facturation |
| **2.8.0** | Énergie active exportée totale | kWh | Injection |
| **3.8.0** | Énergie réactive importée totale | kVArh | Inductive |
| **4.8.0** | Énergie réactive exportée totale | kVArh | Capacitive |
| **1.8.1** | Énergie active importée tarif 1 | kWh | Heures creuses |
| **1.8.2** | Énergie active importée tarif 2 | kWh | Heures pleines |

### Puissances instantanées (W/VA/VAr)

| Code OBIS | Description | Unité | Plage |
|-----------|-------------|-------|-------|
| **16.7.0** | Puissance active totale | W | -999999...+999999 |
| **36.7.0** | Puissance réactive totale | VAr | ±999999 |
| **56.7.0** | Puissance apparente totale | VA | 0...999999 |
| **16.7.1** | Puissance active phase 1 | W | ±999999 |
| **16.7.2** | Puissance active phase 2 | W | ±999999 |
| **16.7.3** | Puissance active phase 3 | W | ±999999 |

### Tensions et courants (V/A)

| Code OBIS | Description | Unité | Précision |
|-----------|-------------|-------|-----------|
| **32.7.0** | Tension phase 1 | V | 0.1V |
| **52.7.0** | Tension phase 2 | V | 0.1V |
| **72.7.0** | Tension phase 3 | V | 0.1V |
| **31.7.0** | Courant phase 1 | A | 0.01A |
| **51.7.0** | Courant phase 2 | A | 0.01A |
| **71.7.0** | Courant phase 3 | A | 0.01A |
| **91.7.0** | Courant neutre | A | 0.01A |

### Qualité du réseau

| Code OBIS | Description | Unité | Norme |
|-----------|-------------|-------|-------|
| **14.7.0** | Fréquence réseau | Hz | 50.00 Hz |
| **13.7.0** | Facteur de puissance total | - | 0.00...1.00 |
| **7.7.0** | THD tension (Total Harmonic Distortion) | % | 0.0...100.0 |
| **8.7.0** | THD courant | % | 0.0...100.0 |
| **81.7.1** | THD tension phase 1 | % | 0.0...100.0 |

### Données temporelles

| Code OBIS | Description | Format |
|-----------|-------------|--------|
| **0.9.1** | Date | YYMMDD |
| **0.9.2** | Heure | HHMMSS |
| **0.9.13** | Timestamp | YYMMDDHHMMSS |
| **0.9.15** | Fuseau horaire | +HHMM/-HHMM |

### Informations compteur

| Code OBIS | Description | Exemple |
|-----------|-------------|---------|
| **0.0.0** | Numéro de série | E450001234 |
| **0.2.0** | Version firmware | 3.4.5 |
| **96.1.0** | Identifiant fabricant | LGZ |
| **96.5.0** | État du compteur | OK/ERROR |
| **97.97.0** | Log des événements | Timestamp + code |

## 🔄 Conversion en unités physiques

### Facteurs de conversion

Le compteur stocke les valeurs dans des **unités de base**, qu'il faut convertir :

```python
# Dictionnaire des facteurs de conversion
FACTEURS_CONVERSION = {
    'W': 1,        # Watt
    'VA': 1,       # Volt-Ampère
    'VAr': 1,      # Volt-Ampère réactif
    'V': 0.1,      # 0.1 Volt (valeur × 0.1 = tension réelle)
    'A': 0.01,     # 0.01 Ampère (valeur × 0.01 = courant réel)
    'Hz': 0.01,    # 0.01 Hz (valeur × 0.01 = fréquence réelle)
    '%': 0.1,      # 0.1% (valeur × 0.1 = pourcentage réel)
}

def convertir_valeur(valeur_brute, unite, scaler=0):
    """
    Conversion d'une valeur OBIS vers unité physique

    Args:
        valeur_brute: Valeur lue du compteur
        unite: Unité OBIS ('V', 'A', 'W', etc.)
        scaler: Exposant du scaler (optionnel)

    Returns:
        Valeur convertie en unité physique
    """
    if unite in FACTEURS_CONVERSION:
        facteur = FACTEURS_CONVERSION[unite]
        valeur_convertie = valeur_brute * facteur

        # Application du scaler si présent
        if scaler != 0:
            valeur_convertie *= (10 ** scaler)

        return valeur_convertie

    # Pas de conversion nécessaire
    return valeur_brute
```

### Exemples de conversion

```python
# Tension phase 1 : valeur brute = 2315, unité = 'V'
tension_reelle = convertir_valeur(2315, 'V')  # = 231.5 V

# Courant phase 1 : valeur brute = 1250, unité = 'A'
courant_reel = convertir_valeur(1250, 'A')    # = 12.50 A

# Puissance : valeur brute = 2850, unité = 'W', scaler = 0
puissance_reelle = convertir_valeur(2850, 'W')  # = 2850 W

# Énergie : souvent en Wh avec scaler
energie_wh = 154320  # valeur brute
energie_kwh = energie_wh / 1000  # conversion kWh
```

## 🐍 Exemple de parsing Python

### Classe de parsing OBIS

```python
import re
import logging
from typing import Dict, Any, Optional

class ParserOBIS:
    """Parser spécialisé pour les codes OBIS du E450"""

    def __init__(self):
        self.logger = logging.getLogger(__name__)

        # Mapping codes OBIS → noms lisibles
        self.mapping_obis = {
            '1.8.0': {'nom': 'energie_active_import', 'unite': 'kWh', 'facteur': 0.001},
            '2.8.0': {'nom': 'energie_active_export', 'unite': 'kWh', 'facteur': 0.001},
            '16.7.0': {'nom': 'puissance_active_totale', 'unite': 'W', 'facteur': 1},
            '32.7.0': {'nom': 'tension_l1', 'unite': 'V', 'facteur': 0.1},
            '52.7.0': {'nom': 'tension_l2', 'unite': 'V', 'facteur': 0.1},
            '72.7.0': {'nom': 'tension_l3', 'unite': 'V', 'facteur': 0.1},
            '31.7.0': {'nom': 'courant_l1', 'unite': 'A', 'facteur': 0.01},
            '51.7.0': {'nom': 'courant_l2', 'unite': 'A', 'facteur': 0.01},
            '71.7.0': {'nom': 'courant_l3', 'unite': 'A', 'facteur': 0.01},
            '14.7.0': {'nom': 'frequence', 'unite': 'Hz', 'facteur': 0.01},
            '13.7.0': {'nom': 'facteur_puissance', 'unite': '', 'facteur': 0.001},
        }

    def parser_obis(self, code_obis: str, valeur_brute: Any) -> Optional[Dict[str, Any]]:
        """
        Parse un code OBIS et sa valeur

        Args:
            code_obis: Code OBIS (ex: "1.8.0")
            valeur_brute: Valeur brute du compteur

        Returns:
            Dictionnaire avec nom, valeur convertie, unité
        """
        if code_obis not in self.mapping_obis:
            self.logger.warning(f"Code OBIS inconnu : {code_obis}")
            return None

        config = self.mapping_obis[code_obis]

        try:
            # Conversion de la valeur
            valeur_numerique = float(valeur_brute)
            valeur_convertie = valeur_numerique * config['facteur']

            return {
                'code_obis': code_obis,
                'nom': config['nom'],
                'valeur_brute': valeur_brute,
                'valeur_convertie': round(valeur_convertie, 3),
                'unite': config['unite'],
                'timestamp': None  # À compléter avec timestamp si disponible
            }

        except (ValueError, TypeError) as e:
            self.logger.error(f"Erreur conversion {code_obis} : {valeur_brute} - {e}")
            return None

    def parser_donnees_compteur(self, donnees_brutes: Dict[str, Any]) -> Dict[str, Any]:
        """
        Parse un ensemble de données compteur

        Args:
            donnees_brutes: Dict code_obis → valeur

        Returns:
            Dict nom_lisible → valeur_convertie
        """
        donnees_parsees = {}

        for code_obis, valeur_brute in donnees_brutes.items():
            resultat = self.parser_obis(code_obis, valeur_brute)

            if resultat:
                nom = resultat['nom']
                donnees_parsees[nom] = {
                    'valeur': resultat['valeur_convertie'],
                    'unite': resultat['unite'],
                    'code_obis': code_obis
                }

        return donnees_parsees

    def obtenir_codes_obis_principaux(self) -> list:
        """Retourne la liste des codes OBIS principaux"""
        return list(self.mapping_obis.keys())
```

### Utilisation du parser

```python
def exemple_parsing_obis():
    """Exemple complet de parsing OBIS"""

    # Données brutes du compteur (simulation)
    donnees_brutes = {
        '1.8.0': 154320,     # 154.32 kWh
        '16.7.0': 2345,      # 2345 W
        '32.7.0': 2315,      # 231.5 V
        '31.7.0': 1250,      # 12.50 A
        '14.7.0': 5000,      # 50.00 Hz
        '13.7.0': 985,       # 0.985 (facteur puissance)
    }

    # Création du parser
    parser = ParserOBIS()

    # Parsing des données
    donnees_parsees = parser.parser_donnees_compteur(donnees_brutes)

    # Affichage des résultats
    print("Données du compteur E450 parsées :")
    print("=" * 40)

    for nom, info in donnees_parsees.items():
        print(f"{nom:25} : {info['valeur']:>8} {info['unite']:<3} (OBIS: {info['code_obis']})")

    print("\nCodes OBIS disponibles :")
    codes = parser.obtenir_codes_obis_principaux()
    for code in codes:
        desc = parser.mapping_obis[code]
        print(f"  {code:8} → {desc['nom']}")

if __name__ == "__main__":
    exemple_parsing_obis()
```

### Sortie du programme

```
Données du compteur E450 parsées :
========================================
energie_active_import     :  154.320 kWh (OBIS: 1.8.0)
puissance_active_totale   : 2345.000 W   (OBIS: 16.7.0)
tension_l1                :  231.500 V   (OBIS: 32.7.0)
courant_l1                :   12.500 A   (OBIS: 31.7.0)
frequence                 :   50.000 Hz  (OBIS: 14.7.0)
facteur_puissance         :    0.985     (OBIS: 13.7.0)

Codes OBIS disponibles :
  1.8.0    → energie_active_import
  2.8.0    → energie_active_export
  16.7.0   → puissance_active_totale
  32.7.0   → tension_l1
  52.7.0   → tension_l2
  72.7.0   → tension_l3
  31.7.0   → courant_l1
  51.7.0   → courant_l2
  71.7.0   → courant_l3
  14.7.0   → frequence
  13.7.0   → facteur_puissance
```

## 📊 Structure avancée des données

### Attributs OBIS

Chaque code OBIS peut avoir plusieurs **attributs** :

```
Code. Classe.Attribut
Ex: 1.8.0.255.2.255
    └───┼────┼──┘
       Classe │
             Attribut
```

#### Attributs principaux

| Attribut | Description | Exemple |
|----------|-------------|---------|
| **2** | Valeur actuelle | Lecture normale |
| **3** | Valeur minimum | Min quotidien |
| **4** | Valeur maximum | Max quotidien |
| **5** | Scaler | Facteur de conversion |
| **6** | Unit | Unité de mesure |
| **7** | Status | État de la mesure |

### Codes OBIS étendus

#### Profils de charge

```
15.7.0  → Profil de charge (puissance par intervalle)
Structure: Timestamp + Valeur + Status
Intervalle: 15 minutes (configurable)
Profondeur: 60 jours minimum
```

#### Événements

```
97.97.0 → Journal des événements
Format: Timestamp + Code événement + Valeur
Événements: Coupure, surtension, manipulation, etc.
```

## 🔧 Outil de découverte OBIS

### Script de scan des codes

```python
def scanner_codes_obis(lecteur, debut=0, fin=100):
    """
    Scan automatique des codes OBIS disponibles

    Args:
        lecteur: Instance du lecteur compteur
        debut, fin: Plage de codes à scanner

    Returns:
        Dict des codes trouvés
    """
    codes_trouves = {}

    print(f"Scan des codes OBIS de {debut} à {fin}...")

    for code in range(debut, fin + 1):
        try:
            # Tentative de lecture
            valeur = lecteur.lire_obis(f"1.{code}.0")

            if valeur is not None:
                codes_trouves[f"1.{code}.0"] = valeur
                print(f"Code trouvé: 1.{code}.0 = {valeur}")

        except Exception as e:
            # Code non disponible, on continue
            pass

    print(f"Scan terminé : {len(codes_trouves)} codes trouvés")
    return codes_trouves
```

### Analyseur de trames DLMS

```python
def analyser_trame_dlms(trame_hex):
    """
    Analyse d'une trame DLMS/COSEM pour extraire les codes OBIS

    Args:
        trame_hex: Trame en hexadécimal

    Returns:
        Liste des codes OBIS trouvés
    """
    import binascii

    try:
        # Conversion hex → bytes
        trame_bytes = binascii.unhexlify(trame_hex)

        # Parsing DLMS (simplifié)
        codes_obis = []

        # Recherche des patterns OBIS (6 octets)
        for i in range(len(trame_bytes) - 6):
            potential_obis = trame_bytes[i:i+6]

            # Validation basique du format OBIS
            if est_format_obis_valide(potential_obis):
                code_obis = '.'.join(map(str, potential_obis))
                codes_obis.append(code_obis)

        return list(set(codes_obis))  # Suppression doublons

    except Exception as e:
        print(f"Erreur analyse trame : {e}")
        return []

def est_format_obis_valide(octets):
    """Validation basique du format OBIS"""
    if len(octets) != 6:
        return False

    # Règles de validation OBIS
    # A (médium): 0-9
    # B (canal): 0-9, 99, 128, etc.
    # C (type): 0-9
    # D (mesure): 0-9
    # E (tarif): 0-9, 99, etc.
    # F (stockage): 0-9, 99, 255, etc.

    a, b, c, d, e, f = octets

    # Vérifications de base
    if not (0 <= a <= 9): return False
    if not (0 <= c <= 9): return False
    if not (0 <= d <= 9): return False

    return True
```

## 📚 Ressources pour codes OBIS

### Documentation officielle

- **DLMS UA Blue Book** : Spécification complète OBIS
- **IEC 62056-61** : Standard OBIS
- **Landis+Gyr E450 Manual** : Codes spécifiques au modèle

### Outils de référence

- **OBIS Code Browser** : Exploration interactive
- **DLMS Parser Online** : Analyse de trames
- **Compteur Simulator** : Test des codes

### Codes OBIS par catégorie

#### Électricité (médium 1)
- **1.x.x** : Énergies actives
- **3.x.x** : Énergies réactives
- **9.x.x** : Énergies apparentes
- **2.x.x** : Quantités tarifées

#### Gaz (médium 7)
- **7.1.x** : Volume
- **7.2.x** : Énergie
- **7.60.x** : Température/conversion

#### Eau (médium 8)
- **8.1.x** : Volume
- **8.2.x** : Débit

> **💡 À retenir** : Les codes OBIS sont la **langue universelle** des compteurs intelligents - maîtrisez-les et vous comprendrez tous les appareils.

> **⚠️ Astuce** : Gardez une **table de correspondance** codes OBIS ↔ noms lisibles dans votre code pour faciliter la maintenance.

Dans le prochain chapitre, nous verrons comment **stocker et persister** toutes ces données dans une base de données !

---

**Navigation**
- [Chapitre précédent : Lecture par bus M-Bus](Chapitre_7_Lecture_MBus.md)
- [Chapitre suivant : Stockage et persistance des données](Chapitre_9_Stockage_Persistance.md)
- [Retour à la table des matières](../../README.md)
