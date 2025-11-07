# 🧭 Chapitre 5 : Structure des menus et affichages LCD

## 📺 L'interface homme-machine du E450

L'afficheur LCD du Landis+Gyr E450 est bien plus qu'un simple écran d'information. C'est une **interface de diagnostic complète** permettant d'accéder à toutes les données du compteur sans équipement externe.

### Caractéristiques de l'afficheur

#### Spécifications techniques

```
Type : LCD reflexif (rétroéclairage LED)
Résolution : 7 segments + symboles (8×2 caractères)
Taille : 70 × 40 mm
Visibilité : 6 mètres (conditions normales)
Température : -20°C à +70°C (fonctionnel)
```

#### Symboles et indicateurs

```
Ligne supérieure : [8 caractères alphanumériques]
Ligne inférieure : [8 caractères numériques/spéciaux]

Symboles disponibles :
┌─────────────────────────────────────────────────┐
│  🔋  │ Batterie RTC faible                       │
│  ⚡  │ Puissance présente                         │
│  🔄  │ Sens du courant (import/export)           │
│  📊  │ Mode test ou programmation                │
│  🔒  │ Accès protégé                             │
│  ⚠️   │ Alerte ou erreur                          │
│  📡  │ Communication en cours                    │
└─────────────────────────────────────────────────┘
```

## 🎮 Système de navigation

### Boutons de commande

Le compteur dispose de **4 boutons poussoirs** en face avant :

```
┌─────────────────────────────────────┐
│              AFFICHEUR              │
│                                     │
│  ┌─────────────────────────────────┐ │
│  │                                 │ │
│  │         CONTENU ECRAN           │ │
│  │                                 │ │
│  └─────────────────────────────────┘ │
│                                     │
│   [◄]     [▲]     [►]     [▼]       │
│  GAUCHE   HAUT   DROITE   BAS       │
└─────────────────────────────────────┘
```

#### Fonctions des boutons

- **◄ GAUCHE** : Navigation arrière / Retour menu
- **▲ HAUT** : Valeur précédente / Scroll up
- **► DROITE** : Validation / Navigation avant
- **▼ BAS** : Valeur suivante / Scroll down

### Modes d'interaction

#### Mode consultation (normal)
- Affichage automatique des informations principales
- Navigation manuelle dans les menus
- Pas de modification possible

#### Mode programmation
- Accessible après authentification
- Modification des paramètres
- Validation requise pour sauvegarde

## 📋 Structure des menus

### Menu principal (affichage par défaut)

Au démarrage, le compteur affiche cycliquement les informations essentielles :

```
Écran 1/4 : Énergie active totale
ENERGIE ACT.
  15432 kWh

Écran 2/4 : Puissance instantanée
PUISSANCE
    2345 W

Écran 3/4 : Tension moyenne
TENSION
    230 V

Écran 4/4 : Heure et date
HEURE
15:42 25/11
```

### Arborescence complète des menus

```
MENU PRINCIPAL
├── 1. ÉNERGIES
│   ├── 1.1 Active importée (1.8.0)
│   ├── 1.2 Active exportée (2.8.0)
│   ├── 1.3 Réactive importée (3.8.0)
│   ├── 1.4 Réactive exportée (4.8.0)
│   └── 1.5 Par tarif (1.8.1, 1.8.2, etc.)
│
├── 2. PUISSANCES
│   ├── 2.1 Active totale (16.7.0)
│   ├── 2.2 Réactive totale (36.7.0)
│   ├── 2.3 Apparente totale (56.7.0)
│   └── 2.4 Par phase (16.7.1, 16.7.2, 16.7.3)
│
├── 3. TENSIONS
│   ├── 3.1 Phase 1 (32.7.0)
│   ├── 3.2 Phase 2 (52.7.0)
│   ├── 3.3 Phase 3 (72.7.0)
│   └── 3.4 Moyenne (13.7.0)
│
├── 4. COURANTS
│   ├── 4.1 Phase 1 (31.7.0)
│   ├── 4.2 Phase 2 (51.7.0)
│   ├── 4.3 Phase 3 (71.7.0)
│   └── 4.4 Neutre (91.7.0)
│
├── 5. QUALITÉ RÉSEAU
│   ├── 5.1 Fréquence (14.7.0)
│   ├── 5.2 Facteur de puissance (13.21.0)
│   ├── 5.3 THD tension (7.7.0)
│   └── 5.4 THD courant (8.7.0)
│
├── 6. HISTORIQUE
│   ├── 6.1 Profils de charge
│   ├── 6.2 Max/min journaliers
│   └── 6.3 Événements
│
├── 7. PARAMÈTRES
│   ├── 7.1 Configuration générale
│   ├── 7.2 Tarification
│   ├── 7.3 Communication
│   └── 7.4 Sécurité
│
└── 8. DIAGNOSTIC
    ├── 8.1 État compteur
    ├── 8.2 Tests automatiques
    ├── 8.3 Versions firmware
    └── 8.4 Logs système
```

## 🔍 Détail des écrans principaux

### Écran d'énergie active (1.8.0)

```
┌─────────────────────────────────────┐
│ ÉNERGIE ACT.                        │
│      15432.567 kWh                  │
│                                     │
│ [◄] Retour   [►] Suivant   [▼] Tarif │
└─────────────────────────────────────┘
```

#### Informations affichées
- **Unité** : kWh (kilowatt-heure)
- **Précision** : 3 décimales (0.001 kWh)
- **Plage** : 0 à 999999.999 kWh
- **Reset** : Via programmation seulement

### Écran de puissance instantanée (16.7.0)

```
┌─────────────────────────────────────┐
│ PUISSANCE                           │
│        2345 W                       │
│                                     │
│ [◄] Retour   [►] Suivant   [▼] Phase │
└─────────────────────────────────────┘
```

#### Caractéristiques
- **Unité** : W (watt)
- **Actualisation** : Toutes les secondes
- **Précision** : 1 W
- **Signe** : + (consommation), - (injection)

### Écran de tension (32.7.0)

```
┌─────────────────────────────────────┐
│ TENSION L1                          │
│       231.5 V                       │
│                                     │
│ [◄] Retour   [►] L2       [▼] L3    │
└─────────────────────────────────────┘
```

#### Détails techniques
- **Unité** : V (volt)
- **Précision** : 0.1 V
- **Fréquence** : 50 Hz
- **Plage** : 0-300 V

## 🔐 Accès aux menus protégés

### Authentification requise

Certains menus nécessitent un **code d'accès** :

```
┌─────────────────────────────────────┐
│ CODE ACCES                           │
│ ********                             │
│                                     │
│ [◄] Annuler  [►] Valider             │
└─────────────────────────────────────┘
```

#### Niveaux d'accès

| Niveau | Code par défaut | Permissions |
|--------|----------------|-------------|
| **Lecteur** | 00000000 | Lecture seule |
| **Opérateur** | 10000000 | Paramètres opérationnels |
| **Configurator** | 20000000 | Configuration complète |
| **Fabricant** | 30000000 | Accès usine |

### Menu de programmation

Une fois authentifié, accès aux paramètres :

```
PARAMETRES
├── Heure/date
├── Adresse M-Bus
├── Tarifs horaires
├── Seuils d'alerte
├── Configuration communication
└── Reset paramètres
```

## 📊 Affichage des données complexes

### Profils de charge

Le compteur stocke des **profils de charge** :

```
PROFIL 15MIN
15:00  2345W
15:15  1890W
15:30  3456W
15:45  2341W
[▲] Prec. [▼] Suiv.
```

#### Caractéristiques
- **Intervalle** : 15 minutes (configurable)
- **Profondeur** : 60 jours minimum
- **Données** : Puissance active moyenne
- **Export** : Via port optique ou M-Bus

### Historique des événements

```
EVENEMENTS
25/11 14:32 COUPURE
25/11 12:15 SURTENSION
24/11 09:45 RESET
[▲] Scroll up
```

#### Types d'événements
- **Coupure secteur** : Timestamp précis
- **Surtension** : Valeur et durée
- **Manipulation** : Ouverture boîtier
- **Communication** : Tentatives d'accès
- **Reset** : Redémarrages système

## ⚙️ Configuration avancée

### Paramétrage des tarifs

```
TARIF 1
08:00-12:00 0.1500
12:00-14:00 0.1800
14:00-18:00 0.1500
[►] Tarif 2
```

#### Possibilités
- **Jusqu'à 8 tarifs** différents
- **Programmation horaire** précise
- **Saisonnière** (été/hiver)
- **Manuelle** ou automatique

### Réglages de communication

```
M-BUS CONFIG
Adresse: 001
Vitesse: 2400
Parité: EVEN
[►] Valider
```

#### Paramètres M-Bus
- **Adresse secondaire** : 001-250
- **Débit** : 300-9600 bauds
- **Parité** : None/Even/Odd
- **Stop bits** : 1 ou 2

## 🔧 Diagnostic et maintenance

### Menu diagnostic

```
DIAGNOSTIC
Tension ref: OK
Courant ref: OK
Memoire: OK
RTC: OK
[►] Tests
```

#### Tests disponibles
- **Auto-test** : Vérification circuits internes
- **Précision** : Comparaison avec référence
- **Communication** : Test des interfaces
- **Mémoire** : Contrôle intégrité EEPROM

### Informations système

```
VERSION
HW: 1.2
FW: 3.4.5
Build: 20231115
Serial: E450001234
```

#### Informations utiles
- **Version hardware** : Révision carte
- **Version firmware** : Logiciel embarqué
- **Date de build** : Compilation
- **Numéro série** : Identifiant unique

## 🎯 Guide pratique d'utilisation

### Consultation des données de base

1. **Alimenter le compteur** (fermer le disjoncteur)
2. **Attendre l'initialisation** (environ 10 secondes)
3. **Observer l'affichage cyclique** automatique
4. **Naviguer avec les boutons** pour explorer

### Accès aux menus avancés

1. **Appuyer sur [►]** depuis l'écran principal
2. **Sélectionner le menu** désiré avec [▲][▼]
3. **Valider avec [►]**
4. **Naviguer dans les sous-menus**

### Programmation sécurisée

1. **Accéder au menu paramètres**
2. **Entrer le code d'accès** (boutons [▲][▼] pour chiffres)
3. **Valider le code**
4. **Modifier les paramètres**
5. **Sauvegarder** (double validation requise)

## 💡 Astuces d'utilisation

### Optimiser la visibilité
- **Orientation** : Afficher face à la lumière
- **Angle** : 90° par rapport à la source lumineuse
- **Distance** : Maximum 6 mètres
- **Température** : Éviter < 0°C (contraste réduit)

### Dépannage courant

#### Écran noir
- Vérifier alimentation secteur
- Contrôler fusibles internes
- Tester batterie RTC si présente

#### Affichage erroné
- Reset paramètres usine (code fabricant)
- Vérification calibration
- Contrôle connexions électriques

#### Boutons non réactifs
- Nettoyer contacts (air comprimé)
- Vérifier câblage interne
- Test continuité boutons

## 🔄 Intégration avec l'application

L'interface LCD est complémentaire à notre application Flask :

```python
def synchroniser_affichage():
    """Synchronisation données LCD ↔ Application"""
    # Lecture via port optique
    donnees_compteur = lecteur_optique.lire_tout()

    # Comparaison avec base de données
    donnees_app = db.get_last_mesure()

    # Détection d'écart
    if abs(donnees_compteur['1.8.0'] - donnees_app.energie_active) > 0.001:
        # Alerte de désynchronisation
        notifier_desync(donnees_compteur, donnees_app)

    return donnees_compteur
```

> **💡 À retenir** : L'afficheur LCD est votre **interface de secours** - même sans ordinateur, vous pouvez toujours consulter les données vitales du compteur.

> **⚠️ Astuce** : Notez toujours les **valeurs affichées** avant toute intervention - cela facilite le diagnostic en cas de problème.

Félicitations ! Vous maîtrisez maintenant les **fondations techniques** du compteur E450. La Partie I est terminée - vous êtes prêt à plonger dans la lecture et l'acquisition des données !

---

**Navigation**
- [Chapitre précédent : Caractéristiques électriques et mécaniques](Chapitre_4_Caracteristiques_Electriques.md)
- [Partie II : Lecture et acquisition des données](../Partie_II_Lecture_Acquisition/)
- [Retour à la table des matières](../../README.md)
