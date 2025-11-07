# 📚 Chapitre 30 : Annexes techniques

## 🔧 Spécifications détaillées E450

### Caractéristiques métrologiques

| Paramètre | Valeur | Norme |
|-----------|--------|-------|
| **Classe de précision** | B | EN 50470-1 |
| **Tension nominale** | 230V ± 20% | IEC 62052-11 |
| **Courant nominal** | 5A / 10A | IEC 62053-21 |
| **Courant max** | 100A | IEC 62053-23 |
| **Fréquence** | 50 Hz ± 5% | EN 50470-3 |
| **Température de fonctionnement** | -25°C à +70°C | IEC 62052-11 |
| **Humidité** | 0-95% (sans condensation) | IEC 62052-11 |

### Interfaces de communication

#### Port optique
```text
• Norme: IEC 62056-21 mode C
• Technologie: Infrarouge
• Débit: 300-9600 bauds
• Protocole: DLMS/COSEM
• Alimentation: Passive
• Sécurité: Isolation galvanique
```

#### Bus M-Bus
```text
• Norme: EN 13757-2/3
• Technologie: RS485 half-duplex
• Débit: 300-9600 bauds
• Topologie: Bus linéaire
• Distance max: 1000m (sans répéteur)
• Alimentation: Parasite 30-42V
• Adressage: 1-250 équipements
```

#### Interface G3-PLC (optionnel)
```text
• Technologie: Power Line Communication
• Fréquence: Bande CENELEC A (35-91kHz)
• Débit: 1-128 kbps
• Portée: 1-2 km (selon réseau)
• Modulation: OFDM
• Robustesse: Très haute (filtrage bruit)
```

## 📊 Table de correspondance codes OBIS

### Énergies cumulées (kWh)

| Code OBIS | Description complète | Type | Unité | Tarif |
|-----------|---------------------|------|-------|-------|
| `1.8.0.255.2.255` | Énergie active importée totale | Import | kWh | Total |
| `1.8.1.255.2.255` | Énergie active importée tarif 1 | Import | kWh | T1 (HP) |
| `1.8.2.255.2.255` | Énergie active importée tarif 2 | Import | kWh | T2 (HC) |
| `2.8.0.255.2.255` | Énergie active exportée totale | Export | kWh | Total |
| `3.8.0.255.2.255` | Énergie réactive importée totale | Réactive | kVArh | Total |
| `4.8.0.255.2.255` | Énergie réactive exportée totale | Réactive | kVArh | Total |
| `9.8.0.255.2.255` | Énergie apparente totale | Apparente | kVAh | Total |

### Puissances instantanées

| Code OBIS | Description complète | Phase | Unité | Remarque |
|-----------|---------------------|-------|-------|----------|
| `16.7.0.255.2.255` | Puissance active totale | Total | W | Puissance consommée |
| `36.7.0.255.2.255` | Puissance réactive totale | Total | VAr | Puissance réactive |
| `56.7.0.255.2.255` | Puissance apparente totale | Total | VA | Puissance apparente |
| `16.7.1.255.2.255` | Puissance active phase 1 | L1 | W | Par phase |
| `16.7.2.255.2.255` | Puissance active phase 2 | L2 | W | Par phase |
| `16.7.3.255.2.255` | Puissance active phase 3 | L3 | W | Par phase |

### Tensions et courants

| Code OBIS | Description complète | Phase | Unité | Plage |
|-----------|---------------------|-------|-------|-------|
| `32.7.0.255.2.255` | Tension phase 1 | L1 | V | 0-300V |
| `52.7.0.255.2.255` | Tension phase 2 | L2 | V | 0-300V |
| `72.7.0.255.2.255` | Tension phase 3 | L3 | V | 0-300V |
| `31.7.0.255.2.255` | Courant phase 1 | L1 | A | 0-100A |
| `51.7.0.255.2.255` | Courant phase 2 | L2 | A | 0-100A |
| `71.7.0.255.2.255` | Courant phase 3 | L3 | A | 0-100A |

### Qualité réseau

| Code OBIS | Description complète | Type | Unité | Seuil normal |
|-----------|---------------------|------|-------|--------------|
| `14.7.0.255.2.255` | Fréquence réseau | Fréquence | Hz | 50 ± 1 Hz |
| `13.21.0.255.2.255` | Facteur de puissance total | Qualité | - | > 0.9 |
| `7.7.0.255.2.255` | THD tension totale | Distorsion | % | < 8% |
| `8.7.0.255.2.255` | THD courant total | Distorsion | % | < 8% |

## 🔌 Schémas de câblage

### Connexion port optique

```
Ordinateur ─── Adaptateur USB-IR ─── Câble optique ─── Port compteur
               (FTDI/CH340)            (IR LED)           (fenêtre IR)
```

**Configuration série** :
```python
{
    'baudrate': 9600,
    'bytesize': serial.EIGHTBITS,
    'parity': serial.PARITY_NONE,
    'stopbits': serial.STOPBITS_ONE,
    'timeout': 5
}
```

### Bus M-Bus (topologie linéaire)

```
Alimentation M-Bus ──┬── Compteur 1 ─── Compteur 2 ─── ... ─── Compteur N
                     │
                     └── Résistance de terminaison (120Ω)
```

**Connexions** :
- **Rouge** : +36V (alimentation)
- **Noir** : GND (masse)
- **Jaune** : Signal A (data+)
- **Vert** : Signal B (data-)

### Configuration réseau Home Assistant

```yaml
# configuration.yaml
mqtt:
  broker: localhost
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password

# Capteurs énergétiques
sensor:
  - platform: mqtt
    name: "Energie Totale"
    state_topic: "homeassistant/sensor/compteur_e450/total"
    value_template: "{{ value_json.energy_kwh }}"
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing

  - platform: mqtt
    name: "Puissance Active"
    state_topic: "homeassistant/sensor/compteur_e450/power"
    value_template: "{{ value_json.power_w }}"
    unit_of_measurement: "W"
    device_class: power
```

## 🐍 Scripts utilitaires

### Lecteur port optique basique

```python
#!/usr/bin/env python3
"""
Lecteur de compteur E450 via port optique
"""
import serial
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LecteurE450:
    def __init__(self, port='/dev/ttyUSB0'):
        self.port = port
        self.config = {
            'baudrate': 300,
            'bytesize': serial.SEVENBITS,
            'parity': serial.PARITY_EVEN,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 5
        }

    def connexion_initiale(self):
        """Établit la connexion initiale"""
        try:
            with serial.Serial(self.port, **self.config) as ser:
                # Mode A - Identification
                ser.write(b'/?!\r\n')
                time.sleep(0.5)
                reponse = ser.read(100)
                logger.info(f"Réponse initiale: {reponse}")

                # Mode C - Communication complète
                ser.write(b'\x06000\r\n')  # ACK + numéro client
                time.sleep(0.5)

                return True
        except Exception as e:
            logger.error(f"Erreur connexion: {e}")
            return False

    def lire_donnees(self, codes_obis):
        """Lit les codes OBIS spécifiés"""
        donnees = {}

        try:
            with serial.Serial(self.port, **self.config) as ser:
                for code in codes_obis:
                    # Requête DLMS/COSEM pour le code
                    requete = f'R1\r\n{code}()\r\n'.encode()
                    ser.write(requete)

                    time.sleep(0.2)
                    reponse = ser.read(200)

                    donnees[code] = self._parser_reponse(reponse)

        except Exception as e:
            logger.error(f"Erreur lecture: {e}")

        return donnees

    def _parser_reponse(self, reponse):
        """Parse la réponse brute du compteur"""
        try:
            texte = reponse.decode('ascii', errors='ignore')
            # Logique de parsing selon format DLMS/COSEM
            # À adapter selon la structure exacte
            return texte.strip()
        except:
            return None
```

### Client M-Bus Python

```python
#!/usr/bin/env python3
"""
Client M-Bus pour réseau de compteurs E450
"""
import serial
import time
from typing import List, Dict, Optional

class ClientMBus:
    def __init__(self, port='/dev/ttyUSB1'):
        self.port = port
        self.config = {
            'baudrate': 9600,
            'bytesize': serial.EIGHTBITS,
            'parity': serial.PARITY_NONE,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 2
        }

    def scanner_reseau(self) -> List[int]:
        """Scanne le réseau M-Bus pour trouver les équipements"""
        equipements = []

        try:
            with serial.Serial(self.port, **self.config) as ser:
                for adresse in range(1, 251):  # 1-250 valides
                    if self._tester_adresse(ser, adresse):
                        equipements.append(adresse)
                        time.sleep(0.1)  # Pause entre tests

        except Exception as e:
            print(f"Erreur scan: {e}")

        return equipements

    def _tester_adresse(self, ser, adresse: int) -> bool:
        """Teste si un équipement répond à une adresse"""
        try:
            # Requête de ping M-Bus
            requete = bytes([0x68, 0x03, 0x03, 0x68])  # En-tête court
            requete += bytes([adresse])  # Adresse
            requete += bytes([0x00, 0x00])  # Contrôle + checksum (simplifié)

            ser.write(requete)
            time.sleep(0.5)

            reponse = ser.read(10)
            return len(reponse) > 0

        except:
            return False

    def lire_compteur(self, adresse: int, codes_obis: List[str]) -> Dict:
        """Lit les données d'un compteur spécifique"""
        donnees = {}

        try:
            with serial.Serial(self.port, **self.config) as ser:
                for code in codes_obis:
                    # Requête M-Bus spécialisée
                    requete = self._construire_requete_mbus(adresse, code)

                    ser.write(requete)
                    time.sleep(1)

                    reponse = ser.read(200)
                    donnees[code] = self._parser_reponse_mbus(reponse)

        except Exception as e:
            print(f"Erreur lecture {adresse}: {e}")

        return donnees

    def _construire_requete_mbus(self, adresse: int, code_obis: str) -> bytes:
        """Construit une requête M-Bus pour un code OBIS"""
        # Implémentation simplifiée - adapter selon spécifications M-Bus
        requete = bytes([0x68, 0x0B, 0x0B, 0x68])  # En-tête long
        requete += bytes([adresse])  # Adresse destination
        # ... logique de construction requête complète
        return requete

    def _parser_reponse_mbus(self, reponse: bytes) -> Optional[float]:
        """Parse une réponse M-Bus"""
        if len(reponse) < 10:
            return None

        try:
            # Logique de décodage M-Bus (BCD, etc.)
            # À adapter selon format exact
            return 0.0  # Valeur temporaire
        except:
            return None
```

## 📋 Check-list déploiement

### Pré-déploiement

- [ ] **Alimentation** : Tension secteur stable (±10%)
- [ ] **Environnement** : Température -25°C à +70°C
- [ ] **Accès** : Port optique accessible, câblage M-Bus prévu
- [ ] **Sécurité** : Protection contre les surtensions
- [ ] **Configuration** : Adresses M-Bus uniques, paramètres réseau

### Tests fonctionnels

- [ ] **Communication** : Port optique et M-Bus opérationnels
- [ ] **Mesures** : Toutes les grandeurs lues correctement
- [ ] **Précision** : Comparaison avec valeurs de référence
- [ ] **Stabilité** : Test de longue durée (24-48h)
- [ ] **Robustesse** : Simulation de perturbations réseau

### Mise en production

- [ ] **Sauvegarde** : Configuration et calibrations sauvegardées
- [ ] **Monitoring** : Système de surveillance opérationnel
- [ ] **Documentation** : Schémas, adresses, paramètres consignés
- [ ] **Formation** : Utilisateurs formés aux procédures
- [ ] **Maintenance** : Plan de maintenance préventive établi

---

**Navigation**
- [Chapitre précédent : Glossaire illustré](Chapitre_29_Glossaire_Illustre.md)
- [Chapitre suivant : Partie VIII - Annexes développeur](../Partie_VIII_Annexes_Developpeur/)
- [Retour à la table des matières](../../README.md)
