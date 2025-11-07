# 🧭 Chapitre 3 : Introduction au compteur Landis+Gyr E450

## 🔍 Qu'est-ce que le Landis+Gyr E450 ?

Le **Landis+Gyr E450** est bien plus qu'un simple compteur électrique. C'est un **ordinateur embarqué spécialisé** dans la mesure et la transmission de données énergétiques, doté d'une intelligence intégrée et de capacités de communication avancées.

### Positionnement dans la gamme Landis+Gyr

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     E350        │    │     E450        │    │     E550        │
│   Monophasé     │    │   Triphasé      │    │   Intelligent   │
│   Basique       │    │   Standard      │    │   Avancé        │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ • Mesure simple │    │ • DLMS/COSEM    │    │ • PLC intégré   │
│ • LCD 7 seg.    │    │ • M-Bus         │    │ • Tarif dynamique│
│ • Port optique  │    │ • Port optique  │    │ • Load control  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
   Remplacement     ←─── NOUS SOMMES ICI ───→   Futur proof
   progressif                  E450
```

## 📐 Caractéristiques physiques

### Dimensions et installation

```
Dimensions extérieures : 170 × 125 × 65 mm (L × H × P)
Poids : ~400g (sans connecteurs)

Installation :
• DIN rail EN 60715 (montage sur rail)
• Fixation murale possible
• Degré de protection : IP51 (frontal)
• Température : -40°C à +70°C
```

### Interfaces de connexion

#### Face avant
```
┌─────────────────────────────────────────┐
│              AFFICHEUR LCD              │
│                                         │
│  ┌─────────────────────────────────────┐ │
│  │                                     │ │
│  │         ÉNERGIE ACTIVE              │ │
│  │                                     │ │
│  │          15 432 kWh                 │ │
│  │                                     │ │
│  └─────────────────────────────────────┘ │
│                                         │
│  BOUTONS : ◄ ▲ ▼ ►                      │
└─────────────────────────────────────────┘
```

#### Face arrière (connecteurs)
```
┌─────────────────────────────────────────┐
│         CONNECTEURS ÉLECTRIQUES         │
├─────────────────────────────────────────┤
│  ┌──┐ ┌──┐ ┌──┐    ┌──┐ ┌──┐ ┌──┐     │
│  │L1│ │L2│ │L3│    │N │ │  │ │  │     │
│  └──┘ └──┘ └──┘    └──┘ └──┘ └──┘     │
│  Phase 1  Phase 2  Phase 3  Neutre    │
├─────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    │
│  │ PORT OPTIQUE│    │   M-BUS     │    │
│  │   (IEC62056)│    │  (RS485)    │    │
│  └─────────────┘    └─────────────┘    │
└─────────────────────────────────────────┘
```

## ⚡ Spécifications électriques

### Plage de mesure

| Paramètre | Valeur | Unité |
|-----------|--------|-------|
| **Tension nominale** | 3 × 230/400 | V |
| **Tension de service** | 0.8...1.15 × Uₙ | V |
| **Fréquence** | 50 | Hz |
| **Courant de base** | 5 | A |
| **Courant maximal** | 100 | A |
| **Courant de démarrage** | 0.002 | A |

### Précision de mesure

```
Classe de précision : B (1% pour I ≥ 0.05 Iₘₐₓ)
Courant de référence : 5 A
Tension de référence : 230 V

Erreurs maximales :
• Énergie active : ±1.0%
• Énergie réactive : ±2.0%
• Puissance : ±1.5%
```

### Consommation propre

```
Consommation en veille : < 2 W / < 10 VA
Consommation en fonctionnement : < 3 W / < 13 VA
Perte par phase : < 0.5 W
```

## 🧠 Intelligence embarquée

### Microcontrôleur

Le E450 embarque un **microcontrôleur 32-bit** avec :
- **Mémoire Flash** : 256 KB pour firmware et données
- **RAM** : 32 KB pour calculs temps réel
- **EEPROM** : 8 KB pour paramètres persistants
- **RTC** : Horloge temps réel avec batterie

### Capacités de calcul

```c
// Exemple simplifié du firmware interne
void calculer_energies(void) {
    // Lecture tensions/courants
    float U[3], I[3];
    lire_tensions_courants(U, I);

    // Calcul puissances instantanées
    float P[3], Q[3];
    for (int phase = 0; phase < 3; phase++) {
        P[phase] = U[phase] * I[phase] * cos(phi[phase]);
        Q[phase] = U[phase] * I[phase] * sin(phi[phase]);
    }

    // Intégration temporelle
    energie_active += P_totale * delta_t;
    energie_reactive += Q_totale * delta_t;
}
```

## 📊 Données disponibles

### Registres principaux

Le compteur maintient en permanence **plus de 100 registres** :

#### Énergies cumulées
- **1.8.0** : Énergie active totale (+)
- **2.8.0** : Énergie active totale (-)
- **3.8.0** : Énergie réactive totale (+)
- **4.8.0** : Énergie réactive totale (-)

#### Puissances instantanées
- **16.7.0** : Puissance active totale (W)
- **36.7.0** : Puissance réactive totale (VAR)
- **56.7.0** : Puissance apparente totale (VA)

#### Qualité réseau
- **32.7.0** : Tension phase 1 (V)
- **52.7.0** : Tension phase 2 (V)
- **72.7.0** : Tension phase 3 (V)
- **31.7.0** : Courant phase 1 (A)

### Historique intégré

Le compteur stocke automatiquement :
- **Profils de charge** : Puissance toutes les 15 minutes
- **Événements** : Coupures, surtensions, manipulations
- **Max/min quotidiens** : Valeurs extrêmes par jour
- **Tarif horaire** : Consommation par tranche horaire

## 🔐 Sécurité et authentification

### Mécanismes de protection

#### Authentification physique
- **Cache de protection** : Accès limité au port optique
- **Scellés** : Détection d'ouverture non autorisée
- **Tamper detection** : Capteurs anti-manipulation

#### Authentification logicielle
- **Mot de passe** : Protection des paramètres (défaut : 00000000)
- **Clé OBIS** : Accès sélectif aux données sensibles
- **Association** : Niveaux d'accès (public, lecteur, configurateur)

#### Sécurité réseau (M-Bus)
- **Adresse secondaire** : Identifiant unique du compteur
- **BACnet** : Authentification par clé
- **Chiffrement** : Données sensibles cryptées

## 📡 Interfaces de communication

### Port optique (IEC 62056-21)

```
Protocole : Mode C (DLMS/COSEM)
Débit : 300 à 9600 bauds
Connecteur : IEC 1107 (infrarouge)
Alimentation : Passive (puissance optique)
```

#### Avantages
- ✅ Connexion directe et fiable
- ✅ Pas de configuration réseau
- ✅ Sécurité physique
- ✅ Standardisé internationalement

#### Limitations
- ❌ Distance limitée (< 1m)
- ❌ Un seul accès simultané
- ❌ Nécessite adaptateur USB

### Bus M-Bus (EN 13757)

```
Topologie : RS485 half-duplex
Débit : 300 à 9600 bauds
Distance : Jusqu'à 1000m
Adresse : 0 à 250 (secondaire)
```

#### Avantages
- ✅ Réseau multi-compteurs
- ✅ Distance importante
- ✅ Faible coût
- ✅ Robuste industriel

#### Limitations
- ❌ Configuration réseau requise
- ❌ Plus complexe à déboguer
- ❌ Consommation légèrement supérieure

### Autres interfaces (optionnelles)

#### G3-PLC (Power Line Communication)
- Communication sur le réseau électrique
- Pas de câblage supplémentaire
- Intégré dans les modèles avancés

#### Zigbee/LoRa (futur)
- Réseaux maillés
- Très basse consommation
- Idéal pour IoT

## 🏭 Cas d'usage typiques

### Résidentiel individuel
- **Suivi consommation** : Facturation précise
- **Optimisation** : Gestion des pics
- **Solaire** : Injection/rétrocession
- **Domotique** : Intégration Home Assistant

### Immeuble collectif
- **Centralisation** : Un point de collecte
- **Facturation** : Répartition équitable
- **Maintenance** : Détection pannes
- **Reporting** : Consommation globale

### Industrie légère
- **Sous-comptage** : Analyse par machine
- **Efficacité** : KPIs énergétiques
- **Alertes** : Surtensions/coupures
- **Historique** : Analyse post-incident

## 🔧 Maintenance et diagnostic

### Indicateurs de santé

Le compteur fournit des **données de diagnostic** :
- **Heures de fonctionnement** : Durée totale
- **Nombre de resets** : Redémarrages
- **État des fusibles** : Protection interne
- **Température interne** : Surchauffe

### Tests automatiques

```python
def diagnostiquer_compteur(compteur):
    """Fonction de diagnostic complet"""
    tests = {
        'communication': tester_connexion(compteur),
        'precision': verifier_precision(compteur),
        'integrite': controler_integrite(compteur),
        'securite': analyser_securite(compteur)
    }

    return {
        'status': 'OK' if all(tests.values()) else 'WARNING',
        'details': tests,
        'recommandations': generer_recommandations(tests)
    }
```

## 📈 Évolutivité

### Mises à jour firmware
- **OTA** : Over-The-Air via M-Bus
- **Sécurisé** : Signature numérique
- **Réversible** : Rollback possible
- **Sans interruption** : Fonctionnement maintenu

### Extensions futures
- **Tarification dynamique** : Réponse à la demande
- **Contrôle de charge** : Gestion de puissance
- **Blockchain** : Traçabilité énergétique
- **IA embarquée** : Analyse locale

## 🎯 Points clés à retenir

> **💡 À retenir** : Le E450 est un **concentré de technologie** : précision suisse, intelligence embarquée, et connectivité moderne pour l'ère de l'IoT énergétique.

> **⚠️ Astuce** : Toujours noter le numéro de série et la version firmware avant toute manipulation - ces informations sont cruciales pour le support.

Dans le prochain chapitre, nous explorerons en détail les **caractéristiques électriques et mécaniques** du compteur pour comprendre comment il mesure avec une telle précision !

---

**Navigation**
- [Chapitre précédent : Vision et aperçu de l'application Flask](Chapitre_2_Vision_Application_Flask.md)
- [Chapitre suivant : Caractéristiques électriques et mécaniques](Chapitre_4_Caracteristiques_Electriques.md)
- [Retour à la table des matières](../README.md)
