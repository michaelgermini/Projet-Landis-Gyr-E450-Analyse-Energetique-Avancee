# ⚙️ Chapitre 6 : Lecture locale du compteur via port optique USB

## 🔌 Comprendre le port optique

Le port optique du Landis+Gyr E450 est votre **porte d'entrée privilégiée** pour accéder aux données du compteur. Cette interface infrarouge standardisée permet une communication directe et sécurisée.

### Caractéristiques techniques du port optique

#### Spécifications physiques

```
Standard : IEC 62056-21 (anciennement IEC 1107)
Type : Communication infrarouge bidirectionnelle
Portée : 0.1 à 1 mètre (distance optimale : 0.2-0.5m)
Angle : ±30° autour de l'axe optique
Alimentation : Passive (pas d'alimentation électrique)
```

#### Avantages de l'interface optique

- ✅ **Sécurité** : Isolation galvanique complète
- ✅ **Simplicité** : Pas de configuration réseau
- ✅ **Fiabilité** : Résistant aux parasites électriques
- ✅ **Standardisé** : Compatible avec tous les outils
- ✅ **Direct** : Accès immédiat sans intermédiaire

## 🛠️ Adaptateurs et pilotes requis

### Matériel nécessaire

#### Adaptateur USB-série recommandé

```
Modèle : FTDI USB-RS232/IR (ex: ELV USB/IR Leser)
Interface : USB 2.0 Full Speed
Chipset : FTDI FT232RL ou équivalent
Connecteur : Mini-DIN 6 broches (IEC 1107)
Alimentation : Bus USB (5V, 100mA)
Prix : ~25-50€
```

#### Alternatives disponibles

| Marque | Modèle | Avantages | Inconvénients |
|--------|--------|-----------|---------------|
| **ELV** | USB/IR Leser | Fiable, logiciel fourni | Prix élevé |
| **RRR** | IR-USB | Bon rapport qualité/prix | Support limité |
| **Chinois** | Generic FTDI | Très économique | Qualité variable |
| **Landis+Gyr** | OEM | Parfaitement compatible | Rare sur le marché |

### Installation des pilotes

#### Windows 10/11

```batch
# Télécharger les pilotes FTDI
# Depuis : https://www.ftdichip.com/Drivers/VCP.htm

# Installation automatique
# Exécuter CDM21228_Setup.exe en tant qu'administrateur

# Vérification de l'installation
# Gestionnaire de périphériques → Ports (COM et LPT)
# Devrait afficher : USB Serial Port (COMx)
```

#### Linux (Ubuntu/Debian)

```bash
# Installation des pilotes FTDI
sudo apt update
sudo apt install linux-modules-extra-$(uname -r)

# Chargement du module
sudo modprobe ftdi_sio
sudo modprobe usbserial

# Règles udev pour accès utilisateur
echo 'SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", MODE="0666"' | sudo tee /etc/udev/rules.d/99-ftdi.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

#### macOS

```bash
# Installation via Homebrew
brew install libusb

# Ou téléchargement direct des pilotes FTDI
# Depuis le site officiel
```

### Vérification de l'installation

```python
import serial.tools.list_ports

def detecter_adaptateur_optique():
    """Détection automatique de l'adaptateur"""
    ports = serial.tools.list_ports.comports()

    for port in ports:
        if 'FTDI' in port.description or 'USB Serial' in port.description:
            print(f"Adaptateur trouvé : {port.device}")
            print(f"Description : {port.description}")
            print(f"Hardware ID : {port.hwid}")
            return port.device

    print("Aucun adaptateur FTDI détecté")
    return None

# Test
port_optique = detecter_adaptateur_optique()
```

## 📡 Protocoles de communication

### Mode C DLMS/COSEM

Le E450 utilise le **mode C** du protocole IEC 62056-21, qui encapsule DLMS/COSEM :

```
Trame mode C :
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│   /     │  Ident  │   (     │  Data   │   )     │
│  Start  │         │  STX    │         │   ETX   │
└─────────┴─────────┴─────────┴─────────┴─────────┘
```

#### Structure détaillée

- **/** : Caractère de démarrage (0x2F)
- **Ident** : Identifiant du compteur (max 32 caractères)
- **(** : Début des données (STX = 0x02)
- **Data** : Données DLMS/COSEM
- **)** : Fin des données (ETX = 0x03)

### Paramètres de communication

```python
CONFIG_OPTICAL = {
    'port': 'COM3',           # ou '/dev/ttyUSB0'
    'baudrate': 300,          # Vitesse initiale
    'bytesize': serial.SEVENBITS,
    'parity': serial.PARITY_EVEN,
    'stopbits': serial.STOPBITS_ONE,
    'timeout': 2,             # Timeout en secondes
    'xonxoff': False,         # Pas de contrôle flux logiciel
    'rtscts': False,          # Pas de contrôle flux matériel
    'dsrdtr': False           # Pas de contrôle modem
}
```

## 🐍 Implémentation Python de base

### Classe de base pour la communication optique

```python
import serial
import time
import logging

class LecteurOptiqueE450:
    """Lecteur de compteur E450 via port optique"""

    def __init__(self, config):
        self.config = config
        self.serial = None
        self.logger = logging.getLogger(__name__)

    def connecter(self):
        """Établir la connexion série"""
        try:
            self.serial = serial.Serial(**self.config)
            self.logger.info(f"Connecté au port {self.config['port']}")
            return True
        except serial.SerialException as e:
            self.logger.error(f"Erreur de connexion : {e}")
            return False

    def deconnecter(self):
        """Fermer la connexion"""
        if self.serial and self.serial.is_open:
            self.serial.close()
            self.logger.info("Déconnecté")

    def envoyer_commande(self, commande, attendre_reponse=True):
        """Envoyer une commande et recevoir la réponse"""
        if not self.serial or not self.serial.is_open:
            raise ConnectionError("Port série non connecté")

        # Envoi de la commande
        self.serial.write(commande.encode('ascii'))
        self.serial.flush()

        if attendre_reponse:
            # Lecture de la réponse
            time.sleep(0.1)  # Petit délai
            reponse = self.serial.read(self.serial.in_waiting)
            return reponse.decode('ascii', errors='ignore')

        return None
```

### Lecture des données de base

```python
def lire_identification(self):
    """Lire l'identification du compteur (mode A)"""
    # Envoi de la commande d'identification
    reponse = self.envoyer_commande('/?!\r\n')

    if reponse:
        # Parsing de la réponse
        lignes = reponse.strip().split('\r\n')
        for ligne in lignes:
            if ligne.startswith('/'):
                # Format: /IDENTIFICATEUR
                identifiant = ligne[1:]
                self.logger.info(f"Compteur identifié : {identifiant}")
                return identifiant

    return None

def lire_donnees_mode_c(self):
    """Lire les données en mode C (DLMS)"""
    # Commande de lecture complète
    commande = '\x2F\x30\x30\x30\x0D\x0A'  # /000\r\n

    # Envoi et lecture
    reponse = self.envoyer_commande(commande)

    if reponse:
        # Parsing du mode C
        return self.parser_mode_c(reponse)

    return {}
```

## 📚 Utilisation de gurux_dlms

### Installation de la bibliothèque

```bash
# Installation via pip
pip install gurux-dlms

# Ou depuis les sources GitHub
git clone https://github.com/Gurux/Gurux.DLMS.Python.git
cd Gurux.DLMS.Python
python setup.py install
```

### Exemple d'utilisation avec gurux_dlms

```python
from gurux_dlms import GXDLMSClient, GXDLMSTranslator
from gurux_dlms.enums import InterfaceType, Authentication, Security
import serial

class LecteurGuruxE450:
    """Lecteur utilisant la bibliothèque Gurux DLMS"""

    def __init__(self, port):
        # Configuration client DLMS
        self.client = GXDLMSClient()
        self.client.interfaceType = InterfaceType.OPTICAL
        self.client.authentication = Authentication.NONE
        self.client.security = Security.NONE

        # Configuration série
        self.port = port
        self.serial = None

    def connecter(self):
        """Connexion avec authentification"""
        try:
            # Ouverture port série
            self.serial = serial.Serial(
                port=self.port,
                baudrate=9600,
                bytesize=serial.EIGHTBITS,
                parity=serial.PARITY_NONE,
                stopbits=serial.STOPBITS_ONE,
                timeout=5
            )

            # Initialisation DLMS
            self.client.media = self.serial

            # Connexion (SNRM request)
            self.client.connect()

            print("Connecté au compteur E450 via DLMS")
            return True

        except Exception as e:
            print(f"Erreur de connexion : {e}")
            return False

    def lire_registre_obis(self, obis_code):
        """Lire un registre OBIS spécifique"""
        try:
            # Création de l'objet OBIS
            obis = GXDLMSObject(obis_code)

            # Lecture de la valeur
            valeur = self.client.read(obis, 2)[0]  # Attribut 2 = valeur

            return valeur

        except Exception as e:
            print(f"Erreur lecture {obis_code} : {e}")
            return None

    def lire_donnees_principales(self):
        """Lire les données principales du compteur"""
        donnees = {}

        # Codes OBIS principaux
        codes_obis = {
            'energie_active_import': '1.8.0.255.2.255',      # Énergie active importée
            'energie_active_export': '2.8.0.255.2.255',      # Énergie active exportée
            'puissance_active': '16.7.0.255.2.255',          # Puissance active totale
            'tension_l1': '32.7.0.255.2.255',                # Tension phase 1
            'courant_l1': '31.7.0.255.2.255',                # Courant phase 1
        }

        for nom, obis in codes_obis.items():
            valeur = self.lire_registre_obis(obis)
            if valeur is not None:
                donnees[nom] = valeur

        return donnees

    def deconnecter(self):
        """Déconnexion propre"""
        if self.client:
            self.client.disconnect()
        if self.serial and self.serial.is_open:
            self.serial.close()
```

### Test de fonctionnement

```python
def test_lecture_gurux():
    """Test complet de lecture avec Gurux"""

    lecteur = LecteurGuruxE450('COM3')

    if lecteur.connecter():
        try:
            # Lecture des données
            donnees = lecteur.lire_donnees_principales()

            print("Données lues du compteur E450 :")
            for cle, valeur in donnees.items():
                print(f"  {cle}: {valeur}")

        finally:
            lecteur.deconnecter()

# Exécution du test
if __name__ == "__main__":
    test_lecture_gurux()
```

## 🔧 Outils logiciels alternatifs

### pySerial pur

Pour les besoins simples, pySerial suffit :

```python
import serial
import time

def lecture_simple_optique(port):
    """Lecture simple via pySerial"""

    config = {
        'port': port,
        'baudrate': 300,
        'bytesize': serial.SEVENBITS,
        'parity': serial.PARITY_EVEN,
        'stopbits': serial.STOPBITS_ONE,
        'timeout': 2
    }

    with serial.Serial(**config) as ser:
        # Commande d'identification
        ser.write(b'/?!\r\n')
        time.sleep(0.5)

        reponse = ser.read(ser.in_waiting)
        print("Réponse brute :", reponse)

        # Parsing basique
        if reponse:
            lignes = reponse.decode('ascii', errors='ignore').split('\r\n')
            for ligne in lignes:
                if ligne.startswith('/'):
                    print(f"Identifiant compteur : {ligne[1:]}")

# Utilisation
lecture_simple_optique('COM3')
```

### Outils en ligne de commande

#### Linux : screen ou minicom

```bash
# Configuration du port série
stty -F /dev/ttyUSB0 300 evenp cs7 -cstopb

# Utilisation de screen pour communication interactive
screen /dev/ttyUSB0 300

# Dans screen, taper : /?![Entrée]
# Le compteur devrait répondre avec son identification
```

#### Windows : PuTTY

1. **Configuration** :
   - Connection type : Serial
   - Serial line : COM3
   - Speed : 300
   - Data bits : 7
   - Stop bits : 1
   - Parity : Even

2. **Connexion** et envoi de `/?!`

## 🔍 Dépannage des problèmes courants

### Problèmes de connexion

#### Port COM non détecté

```python
# Vérification des ports disponibles
import serial.tools.list_ports

ports = list(serial.tools.list_ports.comports())
for port in ports:
    print(f"{port.device}: {port.description}")

# Sous Linux, permissions
sudo usermod -a -G dialout $USER
# Puis redémarrage de session
```

#### Erreur "Permission denied"

**Linux** :
```bash
# Ajout aux groupes système
sudo usermod -a -G dialout $USER
sudo usermod -a -G tty $USER

# Ou règles udev
echo 'KERNEL=="ttyUSB*", MODE="0666"' | sudo tee /etc/udev/rules.d/99-usb-serial.rules
sudo udevadm control --reload-rules
```

**Windows** :
- Exécuter l'IDE en tant qu'administrateur
- Vérifier que les pilotes FTDI sont bien installés

### Problèmes de communication

#### Pas de réponse du compteur

1. **Vérifier l'alignement** : L'émetteur IR doit être face au port optique
2. **Distance optimale** : 10-30 cm
3. **Pas d'obstacle** : Ligne de vue directe
4. **Bonne orientation** : Angle < 30°

#### Réponses erronées

- **Vérifier la parité** : EVEN pour mode initial
- **Baudrate** : Commencer à 300 bauds
- **Timeout** : Au moins 2 secondes
- **Checksum** : Certains modes l'utilisent

### Diagnostic avancé

```python
def diagnostiquer_connexion_optique(port):
    """Diagnostic complet de la connexion optique"""

    resultats = {
        'port_existe': False,
        'port_accessible': False,
        'compteur_repond': False,
        'mode_c_supporte': False,
        'donnees_lues': False
    }

    try:
        # Test 1 : Port existe
        import os
        if os.path.exists(port) or port.startswith('COM'):
            resultats['port_existe'] = True

        # Test 2 : Port accessible
        with serial.Serial(port, 300, timeout=1) as ser:
            resultats['port_accessible'] = True

            # Test 3 : Compteur répond (mode A)
            ser.write(b'/?!\r\n')
            time.sleep(0.5)
            reponse = ser.read(ser.in_waiting)

            if reponse and reponse.startswith(b'/'):
                resultats['compteur_repond'] = True

                # Test 4 : Mode C
                ser.write(b'\x2F\x30\x30\x30\x0D\x0A')
                time.sleep(1)
                reponse_c = ser.read(ser.in_waiting)

                if b'(' in reponse_c and b')' in reponse_c:
                    resultats['mode_c_supporte'] = True
                    resultats['donnees_lues'] = True

    except Exception as e:
        print(f"Erreur diagnostic : {e}")

    return resultats
```

## 📊 Exemple de session complète

### Script de démonstration

```python
#!/usr/bin/env python3
"""
Démonstration complète de lecture optique E450
"""

import serial
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def main():
    # Configuration
    PORT = 'COM3'  # Adapter selon votre système
    TIMEOUT = 2

    logger.info("Démarrage de la lecture optique E450")

    try:
        # Connexion
        with serial.Serial(
            port=PORT,
            baudrate=300,
            bytesize=serial.SEVENBITS,
            parity=serial.PARITY_EVEN,
            stopbits=serial.STOPBITS_ONE,
            timeout=TIMEOUT
        ) as ser:

            logger.info(f"Connecté au port {PORT}")

            # 1. Identification du compteur (mode A)
            logger.info("Lecture identification...")
            ser.write(b'/?!\r\n')
            time.sleep(0.5)

            reponse_id = ser.read(ser.in_waiting)
            if reponse_id:
                identifiant = reponse_id.decode('ascii', errors='ignore').strip()
                logger.info(f"Compteur identifié : {identifiant}")

            # 2. Lecture des données (mode C)
            logger.info("Lecture données mode C...")
            ser.write(b'\x2F\x30\x30\x30\x0D\x0A')  # /000\r\n
            time.sleep(1)

            reponse_data = ser.read(ser.in_waiting)
            if reponse_data:
                logger.info("Données reçues, parsing...")
                # Ici, intégration avec parser DLMS/COSEM
                print("Données brutes :", reponse_data[:200])  # Premier 200 octets

            logger.info("Lecture terminée avec succès")

    except serial.SerialException as e:
        logger.error(f"Erreur série : {e}")
    except Exception as e:
        logger.error(f"Erreur générale : {e}")

if __name__ == "__main__":
    main()
```

> **💡 À retenir** : Le port optique est votre **interface de référence** pour le développement - simple, directe et toujours disponible.

> **⚠️ Astuce** : Commencez toujours par tester la connexion avec le **mode A** (`/?!`) avant d'essayer le mode C DLMS - c'est un excellent diagnostic.

Dans le prochain chapitre, nous explorerons la **lecture par bus M-Bus**, parfaite pour les installations multi-compteurs !

---

**Navigation**
- [Chapitre précédent : Structure des menus et affichages LCD](../Partie_I_Introduction_Fondations/Chapitre_5_Menus_Affichages.md)
- [Chapitre suivant : Lecture par bus M-Bus](Chapitre_7_Lecture_MBus.md)
- [Retour à la table des matières](../../README.md)
