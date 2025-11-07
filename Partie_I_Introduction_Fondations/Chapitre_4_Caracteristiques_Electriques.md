# 🧭 Chapitre 4 : Caractéristiques électriques et mécaniques

## ⚡ Caractéristiques électriques détaillées

Le Landis+Gyr E450 est un **instrument de mesure de précision** capable de quantifier avec exactitude les différents aspects du flux énergétique triphasé.

### Architecture de mesure

#### Circuit de mesure triphasé

```
Réseau triphasé ──┬──► Transformateur tension VT
                  │
                  ├──► Transformateur courant CT1
                  │
                  └──► Transformateur courant CT2
                                       │
                                       ▼
                    ┌─────────────────────────────────────┐
                    │         ÉLECTRONIQUE DE MESURE      │
                    │                                     │
                    │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐ │
                    │  │ ADC │  │ DSP │  │ MCU │  │ RTC │ │
                    │  │     │  │     │  │     │  │     │ │
                    │  └─────┘  └─────┘  └─────┘  └─────┘ │
                    │                                     │
                    └─────────────────────────────────────┘
                                       │
                                       ▼
                         Registres OBIS (1.8.0, 16.7.0, etc.)
```

#### Composants clés

##### 1. Transformateurs de tension (VT)
- **Ratio** : 230V/√3 → 6V (pour mesure)
- **Précision** : Classe 0.5
- **Isolation** : 4kV entre primaire/secondaire
- **Fréquence** : 50 Hz nominale

##### 2. Transformateurs de courant (CT)
- **Ratio** : 100A/5A (exemple typique)
- **Précision** : Classe 0.5S
- **Charge** : < 0.2 VA (faible consommation)
- **Diamètre** : Ø 21mm (standard)

##### 3. Convertisseur analogique-numérique (ADC)
- **Résolution** : 16 bits minimum
- **Fréquence d'échantillonnage** : 4 kHz par phase
- **Plage dynamique** : 1000:1
- **Filtrage** : Anti-aliasing intégré

## 🔬 Théorie de la mesure énergétique

### Grandeurs électriques mesurées

#### Puissance active (W)

La puissance active représente l'énergie réellement consommée :

```
P = U × I × cos(φ)
```

Où :
- **U** : Tension efficace (V)
- **I** : Courant efficace (A)
- **φ** : Déphasage entre U et I
- **cos(φ)** : Facteur de puissance

#### Puissance réactive (VAR)

La puissance réactive est liée aux champs magnétiques :

```
Q = U × I × sin(φ)
```

Elle ne consomme pas d'énergie mais sollicite le réseau.

#### Puissance apparente (VA)

La puissance apparente est la combinaison des deux :

```
S = √(P² + Q²) = U × I
```

### Calculs par phase et totaux

```python
def calculer_puissances(U, I, phi):
    """
    Calcul des puissances pour une phase
    U, I en valeurs efficaces
    phi en radians
    """
    P = U * I * cos(phi)  # Puissance active
    Q = U * I * sin(phi)  # Puissance réactive
    S = sqrt(P**2 + Q**2) # Puissance apparente

    return P, Q, S

def puissance_totale(P1, P2, P3, Q1, Q2, Q3):
    """Puissance triphasée totale"""
    P_total = P1 + P2 + P3
    Q_total = Q1 + Q2 + Q3
    S_total = sqrt(P_total**2 + Q_total**2)

    return P_total, Q_total, S_total
```

## 📏 Précision et classes de mesure

### Normes applicables

Le E450 respecte les normes internationales :

- **EN 50470-1/3** : Compteurs d'énergie électrique
- **IEC 62053-21/23** : Classes de précision
- **IEC 61557-12** : Mesure de la qualité de l'énergie

### Classes de précision

| Classe | Erreur maximale | Usage |
|--------|----------------|-------|
| **A** | ±1.0% | Usage domestique |
| **B** | ±1.0% | Usage général |
| **C** | ±0.5% | Usage industriel |
| **D** | ±0.2% | Usage laboratoire |

Le E450 est classé **B** pour les mesures standards.

### Facteurs influençant la précision

#### Conditions nominales
- **Température** : 23°C ± 2°C
- **Humidité** : 40-60% HR
- **Fréquence** : 50 Hz ± 1%
- **Forme d'onde** : Sinusoïdale pure

#### Erreurs systématiques
- **Linéarité ADC** : < 0.01%
- **Dérive thermique** : < 0.05%/°C
- **Vieillissement** : < 0.1% par an

## 🌡️ Caractéristiques mécaniques

### Boîtier et matériaux

#### Enveloppe externe
- **Matériau** : Polycarbonate renforcé
- **Indice IK** : IK08 (résistance aux chocs)
- **Indice IP** : IP51 frontal, IP20 arrière
- **Couleur** : Gris RAL 7035

#### Dimensions précises

```
Vue de face :
┌─────────────────────────────────────┐
│                                     │
│  Largeur totale : 170 mm            │
│  Hauteur totale : 125 mm            │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Afficheur LCD : 70×40 mm    │   │
│  └─────────────────────────────┘   │
│                                     │
│  Profondeur totale : 65 mm          │
└─────────────────────────────────────┘

Vue latérale :
┌─────────────────┐
│                 │
│  DIN rail       │
│  EN 60715       │
│  (35 mm)        │
│                 │
└─────────────────┘
```

### Système de fixation

#### Montage sur rail DIN
- **Standard** : EN 60715 (35 mm)
- **Fixation** : Ressort à déclenchement
- **Démontage** : Outil isolant plat
- **Vibration** : Résistant jusqu'à 2g

#### Fixation murale (option)
- **Supports** : 4 points de fixation
- **Vis** : M4 × 25 mm
- **Écartement** : 140×95 mm

## 🔌 Connecteurs électriques

### Bornes de puissance

#### Configuration standard (direct)
```
Bornier supérieur (arrivée) :
┌──┬──┬──┬──┬──┬──┐
│L1│L2│L3│N │  │  │
└──┴──┴──┴──┴──┴──┘

Bornier inférieur (sortie) :
┌──┬──┬──┬──┬──┬──┐
│L1│L2│L3│N │  │  │
└──┴──┴──┴──┴──┴──┘

Section des conducteurs : 1.5 - 16 mm²
Couple de serrage : 1.2 - 1.8 Nm
```

#### Configuration avec TC (transformateurs de courant)
```
Bornier TC (secondaire) :
┌──┬──┬──┬──┬──┬──┬──┬──┐
│S1│S2│S1│S2│S1│S2│K │L │
└──┴──┴──┴──┴──┴──┴──┴──┘
```

### Connecteurs de communication

#### Port optique
- **Standard** : IEC 62056-21
- **Type** : Connecteur infrarouge
- **Distance** : < 1 mètre
- **Angle** : ±30° de l'axe

#### Port M-Bus
- **Standard** : EN 13757
- **Type** : Bornes à vis
- **Topologie** : Bus RS485
- **Terminaison** : Résistance 120Ω

## 🛡️ Sécurité électrique

### Protections intégrées

#### Fusibles internes
- **Type** : Fusibles à cartouche 5×20 mm
- **Rating** : 630 mA (lent)
- **Localisation** : Circuit de mesure
- **Remplacement** : Accès par face arrière

#### Protection contre les surtensions
- **Varistances** : MOV 275V
- **Décharge** : Spark-gap
- **Filtrage** : Condensateurs RFI

#### Mise à la terre
- **Connexion** : Borne dédiée
- **Section** : 4 mm² minimum
- **Impédance** : < 1Ω

## 🌡️ Conditions environnementales

### Température de fonctionnement

```
Température ambiante :
• Fonctionnement : -40°C à +70°C
• Stockage : -40°C à +85°C
• Transport : -40°C à +85°C

Classes de température :
• T1 : -10°C à +40°C (usage général)
• T2 : -25°C à +55°C (extérieur protégé)
• T3 : -25°C à +70°C (extérieur)
```

### Humidité et condensation

```
Humidité relative :
• Fonctionnement : 0% à 95% HR (sans condensation)
• Stockage : 0% à 95% HR

Protection contre la condensation :
• Chauffage interne : Résistance 2W
• Activation : < 5°C ou > 80% HR
```

### Vibrations et chocs

```
Vibrations (EN 60068-2-6) :
• Fréquence : 10-150 Hz
• Amplitude : 0.35 mm
• Accélération : 5g

Chocs (EN 60068-2-27) :
• Semi-sinusoïdal : 30g / 11ms
• Axe : 3 directions
```

## 🔧 Maintenance et calibration

### Intervals de vérification

#### Calibration initiale
- **Usine** : ±0.2% de précision
- **Certificat** : Traçable aux étalons nationaux
- **Validité** : Illimitée (si conditions respectées)

#### Vérifications périodiques
- **Légale** : Tous les 10 ans (norme)
- **Recommandée** : Tous les 5 ans
- **Après incident** : Coupure, surtension, manipulation

### Tests de performance

#### Test de précision
```python
def test_precision_compteur(compteur, charge_reference):
    """Test de précision selon norme"""
    erreurs = []

    for charge in [0.05, 0.1, 0.5, 1.0]:  # Multiples du courant de base
        mesure_compteur = compteur.lire_puissance()
        mesure_reference = charge_reference * charge

        erreur = abs(mesure_compteur - mesure_reference) / mesure_reference
        erreurs.append(erreur * 100)  # En %

    return {
        'erreurs_max': max(erreurs),
        'erreurs_moyenne': sum(erreurs)/len(erreurs),
        'conforme_classe_B': max(erreurs) <= 1.0
    }
```

#### Test d'intégrité
- **Auto-test** : Au démarrage
- **Continu** : Surveillance des circuits
- **Rapport** : Via codes OBIS (F.0.0)

## 📊 Données techniques résumées

### Fiche technique compacte

| Caractéristique | Valeur | Unité |
|----------------|--------|-------|
| **Tension nominale** | 3×230/400 | V |
| **Courant nominal** | 5(100) | A |
| **Fréquence** | 50 | Hz |
| **Classe de précision** | B | - |
| **Tension d'impulsion** | 6 | kV |
| **Poids** | 0.4 | kg |
| **MTBF** | 200 000 | h |

> **💡 À retenir** : La précision du E450 vient de son **électronique de mesure sophistiquée** combinant ADC haute résolution, DSP pour les calculs temps réel, et calibration rigoureuse.

> **⚠️ Astuce** : Toujours vérifier la **continuité des conducteurs de terre** avant la mise en service - c'est crucial pour la sécurité et la précision des mesures.

Dans le prochain chapitre, nous explorerons la **structure des menus et affichages LCD** pour comprendre comment interagir physiquement avec le compteur !

---

**Navigation**
- [Chapitre précédent : Introduction au compteur Landis+Gyr E450](Chapitre_3_Introduction_E450.md)
- [Chapitre suivant : Structure des menus et affichages LCD](Chapitre_5_Menus_Affichages.md)
- [Retour à la table des matières](../README.md)
