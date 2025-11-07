# ⚙️ Chapitre 7 : Lecture par bus M-Bus

## 🌐 Comprendre le bus M-Bus

Le M-Bus (Meter-Bus) est un protocole de communication spécialement conçu pour les **compteurs d'énergie** et autres équipements de mesure. Il permet la création de réseaux fiables et économiques pour la collecte de données.

### Caractéristiques du M-Bus

#### Spécifications générales

```
Standard : EN 13757 (Wireless M-Bus) et EN 1434-3 (Wired M-Bus)
Type : Bus série half-duplex RS485
Topologie : Linéaire ou étoile
Distance maximale : 1000 mètres (RS485)
Nombre d'équipements : 250 par segment
Débit : 300 à 9600 bauds
Alimentation : 30-42V DC via le bus (parasite)
```

#### Avantages du M-Bus

- ✅ **Économique** : Un seul câble pour données + alimentation
- ✅ **Robuste** : Résistant aux parasites industriels
- ✅ **Évolutif** : Ajout facile de nouveaux équipements
- ✅ **Standardisé** : Supporté par tous les fabricants
- ✅ **Basse consommation** : Idéal pour équipements distants

## 🔌 Câblage et topologie

### Configuration électrique

#### Alimentation des équipements

```
┌─────────────────┐    ┌─────────────────┐
│   Maître M-Bus  │────│   E450 Slave    │
│  (PC + Convert) │    │   (Adresse 1)  │
│                 │    │                 │
│  30-42V DC ─────┼────┼─► Alimentation │
│                 │    │   + données     │
└─────────────────┘    └─────────────────┘
```

Le M-Bus utilise un système d'**alimentation parasite** :
- **Tension d'alimentation** : 30-42V DC
- **Courant par équipement** : < 1.5 mA (standby)
- **Puissance maximale** : 65 mW par slave
- **Isolation** : 4kV entre bus et terre

### Câblage RS485

#### Schéma de connexion

```
Maître M-Bus                    E450
┌─────────────┐              ┌─────────────┐
│             │              │            │
│  A (Data+) ─┼──────────────┼─► M-Bus A  │
│             │              │            │
│  B (Data-) ─┼──────────────┼─► M-Bus B  │
│             │              │            │
│  +30V ──────┼──────────────┼─► +VB      │
│             │              │            │
│  GND ───────┼──────────────┼─► GND      │
│             │              │            │
└─────────────┘              └─────────────┘
```

#### Types de câbles

| Type | Section | Distance max | Utilisation |
|------|---------|--------------|-------------|
| **Twisté pair** | 0.8 mm² | 1000m | Installation standard |
| **Twisté pair** | 1.5 mm² | 1500m | Longue distance |
| **Quadripolaire** | 0.8 mm² | 500m | Câble dédié |

### Topologies possibles

#### Topologie linéaire (recommandée)

```
Maître ─── Slave1 ─── Slave2 ─── Slave3 ─── Terminaison
```

- **Avantages** : Simple, économique
- **Inconvénients** : Panne d'un slave affecte les suivants

#### Topologie en étoile

```
           ┌── Slave1
           │
Maître ────┼── Slave2
           │
           └── Slave3
```

- **Avantages** : Fiabilité (pas de cascade de pannes)
- **Inconvénients** : Plus de câblage

### Terminaison du bus

#### Résistance de terminaison

```
┌─────────────┐
│  Maître     │
│             │
│  120Ω ─────┼─► Bus A
│             │
│  120Ω ─────┼─► Bus B
│             │
└─────────────┘
```

- **Valeur** : 120 Ω (0.25W)
- **Position** : Extrémités du bus
- **Rôle** : Absorption des réflexions

## 🛠️ Matériel nécessaire

### Convertisseur USB-M-Bus

#### Modèles recommandés

| Modèle | Interface | Alimentation | Prix approximatif |
|--------|-----------|--------------|-------------------|
| **RRR M-Bus Master** | USB | Externe 30V | 80€ |
| **Elspec M-Bus USB** | USB | Externe 30V | 120€ |
| **Generic RS485-USB** | USB-RS485 | Externe | 25€ |
| **Raspberry Pi + shield** | GPIO | Externe 30V | 50€ |

#### Configuration Raspberry Pi

```bash
# Installation des dépendances
sudo apt update
sudo apt install python3-serial python3-pip

# Configuration GPIO (pins 14/15 pour TX/RX)
# Utilisation d'un shield RS485 ou transistors
```

### Alimentation du bus

#### Solution simple

```
┌─────────────────┐
│ Alimentation    │
│ 30-42V DC       │
├─────────────────┤
│ Sortie +30V ────┼─► Bus VB
│ Sortie GND ─────┼─► Bus GND
│                 │
│ Fusible 2A ─────┼─ (protection)
└─────────────────┘
```

#### Caractéristiques de l'alimentation
- **Tension** : 30-42V DC stable
- **Courant** : 0.5A par équipement minimum
- **Régulation** : < 5% de ripple
- **Protection** : Fusible et diode de protection

## 📚 Bibliothèque pymbus

### Installation

```bash
# Installation via pip
pip install pymbus

# Ou depuis GitHub
git clone https://github.com/gebhardm/pymbus.git
cd pymbus
python setup.py install
```

### Utilisation de base

```python
from mbus.MBus import MBus

def connexion_mbus(port='/dev/ttyUSB0', address=1):
    """Connexion à un compteur M-Bus"""

    # Création de l'instance M-Bus
    mbus = MBus(port)

    try:
        # Connexion
        mbus.connect()

        print("Connecté au bus M-Bus")

        # Lecture d'un slave spécifique
        frame = mbus.read(address)

        if frame:
            print(f"Données reçues de l'adresse {address}")
            return frame
        else:
            print(f"Aucun réponse de l'adresse {address}")
            return None

    except Exception as e:
        print(f"Erreur M-Bus : {e}")
        return None

    finally:
        mbus.disconnect()
```

### Lecture des données E450

```python
from mbus.MBusFrame import MBusFrame
from mbus.MBusDataRecord import MBusDataRecord

def parser_donnees_e450(frame):
    """Parsing des données du frame M-Bus E450"""

    donnees = {}

    if frame and hasattr(frame, 'datarecords'):
        for record in frame.datarecords:
            if hasattr(record, 'value') and hasattr(record, 'unit'):
                # Conversion selon le type de donnée
                valeur = record.value
                unite = record.unit

                # Mapping vers noms lisibles
                if record.function == 'INSTANTANEOUS_VALUE':
                    if 'ENERGY' in str(record.type).upper():
                        donnees['energie_active'] = f"{valeur} {unite}"
                    elif 'POWER' in str(record.type).upper():
                        donnees['puissance_active'] = f"{valeur} {unite}"
                    elif 'VOLTAGE' in str(record.type).upper():
                        donnees[f'tension_{record.storage_number}'] = f"{valeur} {unite}"
                    elif 'CURRENT' in str(record.type).upper():
                        donnees[f'courant_{record.storage_number}'] = f"{valeur} {unite}"

    return donnees

def lire_compteur_e450(port, adresse=1):
    """Lecture complète d'un E450 via M-Bus"""

    from mbus.MBus import MBus

    mbus = MBus(port)

    try:
        mbus.connect()

        # Lecture du compteur
        frame = mbus.read(adresse)

        if frame:
            # Parsing des données
            donnees = parser_donnees_e450(frame)

            print("Données du compteur E450 :")
            for cle, valeur in donnees.items():
                print(f"  {cle}: {valeur}")

            return donnees
        else:
            print(f"Pas de réponse du compteur adresse {adresse}")
            return {}

    except Exception as e:
        print(f"Erreur lecture E450 : {e}")
        return {}

    finally:
        mbus.disconnect()
```

## 🔧 Alternative minimalmodbus

### Installation et configuration

```bash
pip install minimalmodbus
```

### Utilisation pour M-Bus

```python
import minimalmodbus

class LecteurMBusMinimal:
    """Lecteur M-Bus utilisant minimalmodbus"""

    def __init__(self, port, adresse=1):
        self.instrument = minimalmodbus.Instrument(port, adresse)

        # Configuration M-Bus spécifique
        self.instrument.serial.baudrate = 2400
        self.instrument.serial.bytesize = 8
        self.instrument.serial.parity = minimalmodbus.serial.PARITY_EVEN
        self.instrument.serial.stopbits = 1
        self.instrument.serial.timeout = 2

        # Mode M-Bus
        self.instrument.mode = minimalmodbus.MODE_RTU  # Proche du M-Bus

    def lire_registre(self, registre, nombre_bytes=2):
        """Lire un registre M-Bus"""
        try:
            valeur = self.instrument.read_register(registre, nombre_bytes)
            return valeur
        except Exception as e:
            print(f"Erreur lecture registre {registre} : {e}")
            return None

    def lire_donnees_e450(self):
        """Lecture des données principales E450"""
        donnees = {}

        # Registres typiques (à adapter selon mapping)
        registres = {
            'energie_active': 0x0000,    # Registre énergie
            'puissance': 0x0002,         # Registre puissance
            'tension_l1': 0x0004,        # Tension phase 1
            'courant_l1': 0x0006,        # Courant phase 1
        }

        for nom, reg in registres.items():
            valeur = self.lire_registre(reg)
            if valeur is not None:
                donnees[nom] = valeur

        return donnees
```

## 📡 Récupération et décodage des trames

### Structure des trames M-Bus

#### Format de base

```
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│Start│ Len │ Add │ Cmd │ Data│ CRC │ Stop│
└─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

- **Start** : 0x68 (104)
- **Len** : Longueur des données
- **Add** : Adresse du slave (0-250)
- **Cmd** : Commande (RSP_UD = 0x08)
- **Data** : Données DLMS/COSEM
- **CRC** : Contrôle d'intégrité
- **Stop** : 0x16 (22)

### Décodage des données

```python
def decoder_trame_mbus(trame_brute):
    """Décodage d'une trame M-Bus complète"""

    if len(trame_brute) < 8:
        return None

    # Vérification du format
    if trame_brute[0] != 0x68 or trame_brute[-1] != 0x16:
        print("Format de trame invalide")
        return None

    # Extraction des champs
    longueur = trame_brute[1]
    adresse = trame_brute[2]
    commande = trame_brute[3]

    # Vérification CRC
    crc_calcule = calculer_crc(trame_brute[4:-2])
    crc_recu = trame_brute[-2]
    if crc_calcule != crc_recu:
        print("Erreur CRC")
        return None

    # Extraction des données
    donnees = trame_brute[4:-2]

    return {
        'adresse': adresse,
        'commande': commande,
        'donnees': donnees,
        'longueur': longueur
    }

def calculer_crc(donnees):
    """Calcul du CRC M-Bus (polynôme 0xA001)"""
    crc = 0xFFFF

    for octet in donnees:
        crc ^= octet
        for _ in range(8):
            if crc & 1:
                crc >>= 1
                crc ^= 0xA001
            else:
                crc >>= 1

    return crc & 0xFF
```

### Parsing des données DLMS/COSEM

```python
from dlms_cosem import cosem

def parser_donnees_dlms(donnees_binaires):
    """Parsing des données DLMS/COSEM encapsulées"""

    try:
        # Décodage COSEM
        objets = cosem.decode(donnees_binaires)

        resultats = {}

        for obj in objets:
            if hasattr(obj, 'logical_name') and hasattr(obj, 'value'):
                # Mapping OBIS vers noms lisibles
                obis_code = '.'.join(map(str, obj.logical_name))

                resultats[obis_code] = {
                    'valeur': obj.value,
                    'unite': getattr(obj, 'unit', ''),
                    'timestamp': getattr(obj, 'timestamp', None)
                }

        return resultats

    except Exception as e:
        print(f"Erreur parsing DLMS : {e}")
        return {}
```

## 🏗️ Architecture logicielle M-Bus

### Classe de gestion M-Bus

```python
import serial
import time
import logging

class GestionnaireMBus:
    """Gestionnaire complet du bus M-Bus"""

    def __init__(self, port, baudrate=2400):
        self.port = port
        self.baudrate = baudrate
        self.serial = None
        self.compteurs = {}  # Cache des compteurs connus

        # Configuration du logger
        self.logger = logging.getLogger(__name__)

    def connecter(self):
        """Connexion au bus M-Bus"""
        try:
            self.serial = serial.Serial(
                port=self.port,
                baudrate=self.baudrate,
                bytesize=serial.EIGHTBITS,
                parity=serial.PARITY_EVEN,
                stopbits=serial.STOPBITS_ONE,
                timeout=2
            )
            self.logger.info(f"Connecté au bus M-Bus sur {self.port}")
            return True
        except Exception as e:
            self.logger.error(f"Erreur connexion M-Bus : {e}")
            return False

    def scanner_bus(self, debut=0, fin=250):
        """Scan du bus pour découvrir les équipements"""
        equipements = []

        self.logger.info(f"Scan du bus de {debut} à {fin}...")

        for adresse in range(debut, fin + 1):
            if self.tester_adresse(adresse):
                equipements.append(adresse)
                self.logger.info(f"Équipement trouvé à l'adresse {adresse}")

        self.logger.info(f"Scan terminé : {len(equipements)} équipements trouvés")
        return equipements

    def tester_adresse(self, adresse):
        """Test de présence d'un équipement"""
        try:
            # Envoi d'une requête ping
            trame_ping = self.construire_trame(adresse, 0x50)  # SND_NKE

            self.serial.write(trame_ping)
            time.sleep(0.1)

            # Attente réponse
            reponse = self.serial.read(1)
            return len(reponse) > 0

        except Exception as e:
            self.logger.debug(f"Pas de réponse adresse {adresse} : {e}")
            return False

    def lire_compteur(self, adresse):
        """Lecture des données d'un compteur"""
        try:
            # Requête de lecture
            trame_req = self.construire_trame(adresse, 0x5B)  # REQ_UD2

            self.serial.write(trame_req)
            time.sleep(0.5)

            # Lecture réponse
            reponse = self.serial.read(256)

            if reponse:
                # Décodage et parsing
                donnees = self.decoder_reponse(reponse)
                return donnees
            else:
                self.logger.warning(f"Pas de réponse de l'adresse {adresse}")
                return None

        except Exception as e:
            self.logger.error(f"Erreur lecture adresse {adresse} : {e}")
            return None

    def construire_trame(self, adresse, commande, donnees=b''):
        """Construction d'une trame M-Bus"""
        longueur = len(donnees) + 1  # +1 pour C

        trame = bytearray()
        trame.append(0x68)  # Start
        trame.append(longueur)  # L
        trame.append(longueur)  # L répété
        trame.append(adresse)  # A
        trame.append(commande)  # C
        trame.extend(donnees)  # Data
        trame.append(self.calculer_crc(trame[1:-1]))  # CRC
        trame.append(0x16)  # Stop

        return trame

    def calculer_crc(self, donnees):
        """Calcul CRC M-Bus"""
        crc = 0xFFFF
        for octet in donnees:
            crc ^= octet
            for _ in range(8):
                if crc & 1:
                    crc = (crc >> 1) ^ 0xA001
                else:
                    crc >>= 1
        return crc & 0xFF

    def decoder_reponse(self, reponse):
        """Décodage basique de la réponse"""
        if len(reponse) < 8:
            return None

        # Vérifications de base
        if reponse[0] != 0x68 or reponse[-1] != 0x16:
            return None

        # Extraction des données (simplifié)
        donnees_brutes = reponse[4:-2]

        # Ici, intégration avec parser DLMS complet
        return {'donnees_brutes': donnees_brutes.hex()}

    def deconnecter(self):
        """Déconnexion propre"""
        if self.serial and self.serial.is_open:
            self.serial.close()
            self.logger.info("Déconnecté du bus M-Bus")
```

### Utilisation du gestionnaire

```python
def exemple_utilisation_mbus():
    """Exemple complet d'utilisation M-Bus"""

    # Configuration
    PORT = '/dev/ttyUSB0'  # Adapter selon votre système
    ADRESSE_E450 = 1

    # Création du gestionnaire
    mbus = GestionnaireMBus(PORT)

    try:
        # Connexion
        if not mbus.connecter():
            print("Impossible de se connecter au bus")
            return

        # Scan des équipements
        equipements = mbus.scanner_bus(1, 10)
        print(f"Équipements trouvés : {equipements}")

        # Lecture du compteur E450
        if ADRESSE_E450 in equipements:
            donnees = mbus.lire_compteur(ADRESSE_E450)
            if donnees:
                print("Données du compteur :")
                print(donnees)
            else:
                print("Erreur de lecture")
        else:
            print(f"Compteur E450 non trouvé à l'adresse {ADRESSE_E450}")

    finally:
        mbus.deconnecter()

if __name__ == "__main__":
    exemple_utilisation_mbus()
```

## 🔍 Dépannage M-Bus

### Problèmes courants

#### Pas de communication

1. **Alimentation** : Vérifier 30-42V sur le bus
2. **Adressage** : Adresse secondaire correcte (1-250)
3. **Polarité** : A/B correctement connectés
4. **Terminaison** : Résistances 120Ω aux extrémités

#### Erreurs CRC

- **Bruit** : Blindage insuffisant du câble
- **Longueur** : Distance excessive (>1000m)
- **Réflexions** : Terminaison manquante
- **Interférences** : Proximité de sources HF

### Outils de diagnostic

```python
def diagnostiquer_bus_mbus(port):
    """Diagnostic complet du bus M-Bus"""

    diagnostiques = {
        'connexion_physique': False,
        'alimentation_bus': False,
        'equipements_detectes': [],
        'qualite_signal': 'UNKNOWN',
        'erreurs_crc': 0
    }

    try:
        # Test 1 : Connexion physique
        with serial.Serial(port, 2400, timeout=1) as ser:
            diagnostiques['connexion_physique'] = True

            # Test 2 : Présence d'équipements
            for adresse in range(1, 11):  # Test 10 premières adresses
                # Envoi ping
                trame_ping = bytes([0x68, 0x01, 0x01, adresse, 0x50, 0x00, 0x16])
                ser.write(trame_ping)

                time.sleep(0.1)
                reponse = ser.read(1)

                if reponse:
                    diagnostiques['equipements_detectes'].append(adresse)

            # Test 3 : Qualité du signal (simulation)
            if len(diagnostiques['equipements_detectes']) > 0:
                diagnostiques['qualite_signal'] = 'GOOD'
            else:
                diagnostiques['qualite_signal'] = 'WEAK'

    except Exception as e:
        print(f"Erreur diagnostic : {e}")

    return diagnostiques
```

## 📊 Comparaison Port optique vs M-Bus

| Aspect | Port optique | M-Bus |
|--------|--------------|-------|
| **Installation** | Très simple | Nécessite câblage |
| **Coût** | ~25€ (adaptateur) | ~80€ (interface) + câble |
| **Distance** | < 1m | 1000m |
| **Multi-équipements** | 1 seul | 250 max |
| **Vitesse** | Variable | 300-9600 bauds |
| **Alimentation** | Passive | Requise (30V) |
| **Robustesse** | Excellente | Bonne (industriel) |
| **Configuration** | Aucune | Adressage requis |

> **💡 À retenir** : Le M-Bus est idéal pour les **installations collectives** où plusieurs compteurs doivent être interrogés automatiquement.

> **⚠️ Astuce** : Commencez toujours par **scanner le bus** pour identifier les adresses des équipements avant de programmer vos lectures.

Dans le prochain chapitre, nous plongerons dans la **structure des données OBIS** pour comprendre comment décoder les informations du compteur !

---

**Navigation**
- [Chapitre précédent : Lecture locale du compteur via port optique USB](Chapitre_6_Lecture_Port_Optique.md)
- [Chapitre suivant : Structure des données OBIS](Chapitre_8_Structure_OBIS.md)
- [Retour à la table des matières](../../README.md)
