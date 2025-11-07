# 🔌 Chapitre 14 : Lecture réelle du compteur

## ⚡ Configuration port série optique

Maintenant que nous avons les bases théoriques, il est temps de connecter réellement notre application au compteur E450. Commençons par la configuration matérielle et logicielle.

### Vérification du matériel

#### 1. Test de l'adaptateur USB

```python
# scripts/test_adaptateur.py
#!/usr/bin/env python3
"""
Test de l'adaptateur USB port optique
"""

import serial
import serial.tools.list_ports
import sys

def detecter_adaptateurs():
    """Détection de tous les adaptateurs série"""
    ports = serial.tools.list_ports.comports()
    adaptateurs = []

    print("Adaptateurs série détectés :")
    print("=" * 50)

    for port in ports:
        info = {
            'port': port.device,
            'description': port.description,
            'hardware_id': port.hwid,
            'manufacturer': port.manufacturer,
            'serial_number': port.serial_number
        }

        # Identification des adaptateurs potentiels
        if any(keyword in (port.description or '').upper() for keyword in
               ['FTDI', 'USB SERIAL', 'RS232', 'TTL', 'UART']):
            info['type'] = 'adaptateur_serie'
            adaptateurs.append(info)
        elif 'BLUETOOTH' in (port.description or '').upper():
            info['type'] = 'bluetooth'
        elif 'MODEM' in (port.description or '').upper():
            info['type'] = 'modem'
        else:
            info['type'] = 'inconnu'

        print(f"Port: {info['port']}")
        print(f"  Description: {info['description']}")
        print(f"  Hardware ID: {info['hardware_id']}")
        print(f"  Type estimé: {info['type']}")
        print()

    return adaptateurs

def tester_connexion(port, config=None):
    """Test de connexion à un port série"""
    if config is None:
        config = {
            'baudrate': 9600,
            'bytesize': serial.EIGHTBITS,
            'parity': serial.PARITY_NONE,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 2
        }

    try:
        print(f"Test de connexion au port {port}...")
        with serial.Serial(port, **config) as ser:
            print("✅ Connexion établie")
            print(f"  Port: {ser.port}")
            print(f"  Baudrate: {ser.baudrate}")
            print(f"  Bytesize: {ser.bytesize}")
            print(f"  Parity: {ser.parity}")
            print(f"  Stopbits: {ser.stopbits}")

            # Test d'écriture/lecture
            test_data = b"Hello Counter!"
            ser.write(test_data)
            print(f"✅ Écriture de {len(test_data)} octets")

            # Tentative de lecture (timeout court)
            ser.timeout = 0.1
            response = ser.read(100)
            if response:
                print(f"✅ Lecture de {len(response)} octets: {response}")
            else:
                print("ℹ️  Pas de données reçues (normal pour test)")

            return True

    except serial.SerialException as e:
        print(f"❌ Erreur de connexion: {e}")
        return False
    except Exception as e:
        print(f"❌ Erreur inattendue: {e}")
        return False

def test_optique_mode_a(port):
    """Test spécifique du mode A (protocole optique)"""
    try:
        print(f"\nTest du mode A sur {port}...")

        config = {
            'baudrate': 300,
            'bytesize': serial.SEVENBITS,
            'parity': serial.PARITY_EVEN,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 5
        }

        with serial.Serial(port, **config) as ser:
            # Commande d'identification IEC 62056-21 mode A
            commande = b'/?!\r\n'

            print(f"Envoi de la commande: {commande}")
            ser.write(commande)

            # Lecture de la réponse
            response = ser.read(200)
            print(f"Réponse brute: {response}")

            if response:
                try:
                    response_str = response.decode('ascii', errors='ignore')
                    print(f"Réponse décodée: {repr(response_str)}")

                    # Analyse de la réponse
                    lignes = response_str.strip().split('\r\n')
                    for ligne in lignes:
                        if ligne.startswith('/'):
                            print(f"✅ Identifiant compteur détecté: {ligne[1:]}")
                            return ligne[1:]
                        elif 'ERR' in ligne.upper():
                            print(f"❌ Erreur compteur: {ligne}")
                        else:
                            print(f"ℹ️  Ligne reçue: {repr(ligne)}")

                except UnicodeDecodeError:
                    print("❌ Erreur de décodage (données binaires?)")
                    print(f"Données hex: {response.hex()}")

            else:
                print("❌ Aucune réponse reçue")

    except Exception as e:
        print(f"❌ Erreur test mode A: {e}")

    return None

def main():
    """Fonction principale de test"""
    print("🔍 Test des adaptateurs USB pour compteur E450")
    print("=" * 60)

    # Détection
    adaptateurs = detecter_adaptateurs()

    if not adaptateurs:
        print("❌ Aucun adaptateur série détecté!")
        print("\nSolutions possibles:")
        print("1. Vérifiez que l'adaptateur est bien branché")
        print("2. Installez les pilotes FTDI (https://www.ftdichip.com)")
        print("3. Redémarrez votre ordinateur")
        print("4. Essayez un autre port USB")
        sys.exit(1)

    # Tests de connexion
    adaptateurs_optique = [a for a in adaptateurs if a['type'] == 'adaptateur_serie']

    for adaptateur in adaptateurs_optique:
        port = adaptateur['port']
        print(f"\n{'='*20} Test du port {port} {'='*20}")

        # Test de base
        if not tester_connexion(port):
            continue

        # Test spécifique optique
        identifiant = test_optique_mode_a(port)

        if identifiant:
            print(f"\n🎉 SUCCÈS! Compteur E450 détecté: {identifiant}")
            print(f"Port à utiliser dans la configuration: {port}")
            return port
        else:
            print("❌ Ce port ne semble pas connecté à un compteur E450")

    print("\n❌ Aucun compteur E450 détecté sur les ports testés")
    print("\nVérifications à faire:")
    print("1. Le compteur est-il alimenté?")
    print("2. L'adaptateur optique est-il bien positionné?")
    print("3. Y a-t-il de la lumière IR visible?")
    print("4. Essayez de tourner légèrement l'adaptateur")

    return None

if __name__ == "__main__":
    port_trouve = main()
    if port_trouve:
        print(f"\n💡 Utilisez ce port dans votre configuration: {port_trouve}")
    else:
        sys.exit(1)
```

#### 2. Configuration dans l'application

```python
# app/config.py (extension pour la configuration détectée)
def detecter_port_optique():
    """Détection automatique du port optique"""
    import serial.tools.list_ports

    ports = serial.tools.list_ports.comports()

    # Recherche d'un adaptateur FTDI
    for port in ports:
        if 'FTDI' in port.description.upper() or 'USB Serial' in port.description.upper():
            return port.device

    return None

class Config:
    # ...

    # Détection automatique du port optique
    PORT_OPTICAL = os.environ.get('PORT_OPTICAL') or detecter_port_optique() or 'COM3'

    # Configuration validée pour E450
    OPTICAL_CONFIG = {
        'port': PORT_OPTICAL,
        'baudrate': 300,  # Mode A initial
        'bytesize': serial.SEVENBITS,
        'parity': serial.PARITY_EVEN,
        'stopbits': serial.STOPBITS_ONE,
        'timeout': 5,
        'xonxoff': False,
        'rtscts': False,
        'dsrdtr': False
    }
```

## 📡 Script Python de lecture continue

### Service de lecture automatisée

```python
# app/services/lecture_compteur.py
import serial
import time
import threading
import logging
from datetime import datetime, timedelta
from app.extensions import db, scheduler
from app.models import MesureEnergie, EvenementCompteur
from app.services import detecteur_anomalies

logger = logging.getLogger(__name__)

class ServiceLectureCompteur:
    """Service de lecture continue du compteur E450"""

    def __init__(self, config_compteur):
        self.config = config_compteur
        self.serial_conn = None
        self.is_running = False
        self.thread_lecture = None
        self.derniere_lecture = None
        self.intervalle_lecture = 300  # 5 minutes par défaut

        # Statistiques
        self.stats = {
            'lectures_total': 0,
            'lectures_succes': 0,
            'lectures_erreur': 0,
            'derniere_succes': None,
            'derniere_erreur': None
        }

    def demarrer(self):
        """Démarrage du service de lecture"""
        if self.is_running:
            logger.warning("Service déjà démarré")
            return

        logger.info("Démarrage du service de lecture compteur")
        self.is_running = True

        # Démarrage du thread de lecture
        self.thread_lecture = threading.Thread(target=self._boucle_lecture, daemon=True)
        self.thread_lecture.start()

        # Planification des lectures périodiques
        self._planifier_lectures()

    def arreter(self):
        """Arrêt du service de lecture"""
        logger.info("Arrêt du service de lecture compteur")
        self.is_running = False

        if self.thread_lecture and self.thread_lecture.is_alive():
            self.thread_lecture.join(timeout=5)

        # Fermeture de la connexion série
        self._fermer_connexion()

    def _boucle_lecture(self):
        """Boucle principale de lecture"""
        logger.info("Boucle de lecture démarrée")

        while self.is_running:
            try:
                # Tentative de lecture
                donnees = self.lire_donnees_compteur()

                if donnees:
                    # Sauvegarde en base
                    self._sauvegarder_mesure(donnees)
                    self.stats['lectures_succes'] += 1
                    self.stats['derniere_succes'] = datetime.utcnow()

                    logger.info(f"Lecture réussie: {donnees.get('puissance_active_totale', 'N/A')}W")

                else:
                    self.stats['lectures_erreur'] += 1
                    self.stats['derniere_erreur'] = datetime.utcnow()
                    logger.warning("Échec de lecture du compteur")

                self.stats['lectures_total'] += 1

            except Exception as e:
                logger.error(f"Erreur dans la boucle de lecture: {e}")
                self.stats['lectures_erreur'] += 1

            # Attente avant la prochaine lecture
            time.sleep(self.intervalle_lecture)

        logger.info("Boucle de lecture arrêtée")

    def lire_donnees_compteur(self):
        """Lecture des données du compteur"""
        try:
            # Ouverture de la connexion si nécessaire
            if not self.serial_conn or not self.serial_conn.is_open:
                self._ouvrir_connexion()

            if not self.serial_conn:
                return None

            # Lecture selon le protocole
            donnees = self._lecture_protocole_optique()

            if donnees:
                # Ajout de métadonnées
                donnees['timestamp'] = datetime.utcnow()
                donnees['source'] = 'port_optique'
                donnees['qualite'] = 'good'

                self.derniere_lecture = donnees['timestamp']

            return donnees

        except Exception as e:
            logger.error(f"Erreur lecture compteur: {e}")
            return None

    def _ouvrir_connexion(self):
        """Ouverture de la connexion série"""
        try:
            if self.serial_conn and self.serial_conn.is_open:
                self.serial_conn.close()

            self.serial_conn = serial.Serial(**self.config)
            logger.info(f"Connexion série ouverte: {self.config['port']}")

            # Petit délai pour stabilisation
            time.sleep(0.5)

        except serial.SerialException as e:
            logger.error(f"Impossible d'ouvrir la connexion série: {e}")
            self.serial_conn = None

    def _fermer_connexion(self):
        """Fermeture de la connexion série"""
        if self.serial_conn and self.serial_conn.is_open:
            self.serial_conn.close()
            logger.info("Connexion série fermée")

    def _lecture_protocole_optique(self):
        """Lecture selon le protocole optique IEC 62056-21"""
        if not self.serial_conn:
            return None

        try:
            # Phase 1: Mode A - Identification
            self.serial_conn.write(b'/?!\r\n')
            time.sleep(0.2)

            reponse_id = self.serial_conn.read(100)
            if not reponse_id or not reponse_id.startswith(b'/'):
                logger.warning("Pas de réponse d'identification")
                return None

            # Phase 2: Mode C - Données complètes
            commande_data = b'\x2F\x30\x30\x30\x0D\x0A'  # /000\r\n
            self.serial_conn.write(commande_data)
            time.sleep(1)  # Délai plus long pour les données

            reponse_data = self.serial_conn.read(1000)

            if reponse_data:
                # Parsing des données
                return self._parser_reponse_optique(reponse_data)
            else:
                logger.warning("Pas de données reçues en mode C")
                return None

        except serial.SerialException as e:
            logger.error(f"Erreur série pendant la lecture: {e}")
            # Tentative de réouverture
            self._fermer_connexion()
            return None

    def _parser_reponse_optique(self, reponse_brute):
        """Parsing de la réponse optique"""
        try:
            # Conversion en string
            reponse_str = reponse_brute.decode('ascii', errors='ignore')

            # Recherche du bloc de données (entre STX et ETX)
            start_idx = reponse_str.find('\x02')  # STX
            end_idx = reponse_str.find('\x03')    # ETX

            if start_idx == -1 or end_idx == -1:
                logger.warning("Bloc de données non trouvé")
                return None

            bloc_donnees = reponse_str[start_idx+1:end_idx]

            # Parsing des lignes OBIS
            donnees = {}
            lignes = bloc_donnees.split('\r\n')

            for ligne in lignes:
                if '(' in ligne and ')' in ligne:
                    # Format: OBIS(value)
                    obis_code, valeur_str = ligne.split('(', 1)
                    valeur_str = valeur_str.rstrip(')')

                    # Conversion selon le type
                    valeur = self._convertir_valeur_obis(obis_code, valeur_str)

                    if valeur is not None:
                        # Mapping vers noms lisibles
                        nom_lisible = self._map_obis_vers_nom(obis_code)
                        if nom_lisible:
                            donnees[nom_lisible] = valeur

            return donnees if donnees else None

        except Exception as e:
            logger.error(f"Erreur parsing réponse: {e}")
            return None

    def _convertir_valeur_obis(self, obis_code, valeur_str):
        """Conversion d'une valeur OBIS selon son type"""
        try:
            # Suppression des espaces
            valeur_str = valeur_str.strip()

            # Codes OBIS d'énergie (finissent par kWh)
            if obis_code in ['1.8.0', '2.8.0', '1.8.1', '1.8.2']:
                return float(valeur_str) * 1000  # Conversion kWh → Wh

            # Codes OBIS de puissance (W)
            elif obis_code in ['16.7.0', '16.7.1', '16.7.2', '16.7.3']:
                return float(valeur_str)

            # Codes OBIS de tension (V)
            elif obis_code in ['32.7.0', '52.7.0', '72.7.0']:
                return float(valeur_str) * 0.1  # ×0.1 pour précision

            # Codes OBIS de courant (A)
            elif obis_code in ['31.7.0', '51.7.0', '71.7.0']:
                return float(valeur_str) * 0.01  # ×0.01 pour précision

            # Fréquence (Hz)
            elif obis_code == '14.7.0':
                return float(valeur_str) * 0.01

            # Facteur de puissance
            elif obis_code == '13.7.0':
                return float(valeur_str) * 0.001

            else:
                # Valeur numérique générique
                return float(valeur_str)

        except (ValueError, TypeError):
            logger.debug(f"Impossible de convertir {obis_code}: {valeur_str}")
            return None

    def _map_obis_vers_nom(self, obis_code):
        """Mapping des codes OBIS vers noms de champs"""
        mapping = {
            '1.8.0': 'energie_active_import',
            '2.8.0': 'energie_active_export',
            '16.7.0': 'puissance_active_totale',
            '32.7.0': 'tension_l1',
            '52.7.0': 'tension_l2',
            '72.7.0': 'tension_l3',
            '31.7.0': 'courant_l1',
            '51.7.0': 'courant_l2',
            '71.7.0': 'courant_l3',
            '14.7.0': 'frequence',
            '13.7.0': 'facteur_puissance',
        }

        return mapping.get(obis_code)

    def _sauvegarder_mesure(self, donnees):
        """Sauvegarde d'une mesure en base de données"""
        try:
            # Création de l'objet MesureEnergie
            mesure = MesureEnergie(
                energie_active_import=donnees.get('energie_active_import'),
                energie_active_export=donnees.get('energie_active_export'),
                puissance_active_totale=donnees.get('puissance_active_totale'),
                tension_l1=donnees.get('tension_l1'),
                tension_l2=donnees.get('tension_l2'),
                tension_l3=donnees.get('tension_l3'),
                courant_l1=donnees.get('courant_l1'),
                courant_l2=donnees.get('courant_l2'),
                courant_l3=donnees.get('courant_l3'),
                frequence=donnees.get('frequence'),
                facteur_puissance=donnees.get('facteur_puissance'),
                source=donnees.get('source', 'port_optique'),
                qualite=donnees.get('qualite', 'good'),
            )

            db.session.add(mesure)
            db.session.commit()

            logger.debug(f"Mesure sauvegardée: {mesure.id}")

            # Détection d'anomalies
            try:
                anomalies = detecteur_anomalies.detecter_anomalie_mesure(mesure)
                if anomalies:
                    for anomalie in anomalies:
                        self._creer_evenement_anomalie(anomalie, mesure)
            except Exception as e:
                logger.error(f"Erreur détection anomalies: {e}")

        except Exception as e:
            db.session.rollback()
            logger.error(f"Erreur sauvegarde mesure: {e}")
            raise

    def _creer_evenement_anomalie(self, anomalie, mesure):
        """Création d'un événement d'anomalie"""
        try:
            evenement = EvenementCompteur(
                type_evenement=anomalie['type'],
                severite=anomalie['severite'],
                categorie='mesure',
                titre=anomalie['titre'],
                description=anomalie['description'],
                valeur_mesuree=mesure.puissance_active_totale,
                unite='W',
                seuil_declencheur=anomalie.get('seuil'),
            )

            db.session.add(evenement)
            db.session.commit()

            logger.info(f"Événement anomalie créé: {anomalie['titre']}")

        except Exception as e:
            logger.error(f"Erreur création événement: {e}")

    def _planifier_lectures(self):
        """Planification des lectures automatiques"""
        @scheduler.task('interval', id='lecture_compteur_auto',
                       minutes=self.intervalle_lecture // 60, max_instances=1)
        def lecture_automatique():
            # Cette fonction sera appelée par le scheduler
            # La vraie logique est dans le thread
            pass

        logger.info(f"Lectures automatiques planifiées tous les {self.intervalle_lecture}s")

    def obtenir_statistiques(self):
        """Retourne les statistiques du service"""
        return {
            'is_running': self.is_running,
            'derniere_lecture': self.derniere_lecture,
            'intervalle': self.intervalle_lecture,
            'stats': self.stats.copy()
        }

    def forcer_lecture(self):
        """Force une lecture immédiate"""
        logger.info("Lecture forcée demandée")

        try:
            donnees = self.lire_donnees_compteur()
            if donnees:
                self._sauvegarder_mesure(donnees)
                return True, donnees
            else:
                return False, None
        except Exception as e:
            logger.error(f"Erreur lecture forcée: {e}")
            return False, str(e)
```

### Intégration dans l'application Flask

```python
# app/services/__init__.py (extension)
from .lecture_compteur import ServiceLectureCompteur

# Instance globale
service_lecture = None

def init_services(app):
    """Initialisation des services au démarrage de l'app"""
    global service_lecture

    config_optique = app.config.get('OPTICAL_CONFIG')
    if config_optique:
        service_lecture = ServiceLectureCompteur(config_optique)

        # Démarrage automatique si configuré
        if app.config.get('AUTO_START_LECTURE', True):
            service_lecture.demarrer()

# app/__init__.py (extension)
def create_app(config_name=None):
    # ...
    from app.services import init_services
    init_services(app)
    # ...

# Route pour contrôler le service
@main_bp.route('/lecture/status')
@login_required
def status_lecture():
    """Status du service de lecture"""
    from app.services import service_lecture

    if service_lecture:
        stats = service_lecture.obtenir_statistiques()
        return jsonify({
            'status': 'active' if stats['is_running'] else 'inactive',
            'stats': stats
        })
    else:
        return jsonify({'status': 'non_configuré'})

@main_bp.route('/lecture/start', methods=['POST'])
@login_required
def demarrer_lecture():
    """Démarrage manuel du service de lecture"""
    from app.services import service_lecture

    if service_lecture and not service_lecture.is_running:
        service_lecture.demarrer()
        return jsonify({'success': True, 'message': 'Service démarré'})
    else:
        return jsonify({'success': False, 'message': 'Service déjà démarré ou non configuré'})

@main_bp.route('/lecture/stop', methods=['POST'])
@login_required
def arreter_lecture():
    """Arrêt manuel du service de lecture"""
    from app.services import service_lecture

    if service_lecture and service_lecture.is_running:
        service_lecture.arreter()
        return jsonify({'success': True, 'message': 'Service arrêté'})
    else:
        return jsonify({'success': False, 'message': 'Service déjà arrêté ou non configuré'})

@main_bp.route('/lecture/test', methods=['POST'])
@login_required
def test_lecture():
    """Test de lecture immédiat"""
    from app.services import service_lecture

    if service_lecture:
        success, result = service_lecture.forcer_lecture()

        if success:
            return jsonify({
                'success': True,
                'message': 'Lecture réussie',
                'data': result
            })
        else:
            return jsonify({
                'success': False,
                'message': f'Échec de la lecture: {result}'
            })
    else:
        return jsonify({'success': False, 'message': 'Service non configuré'})
```

### Démarrage automatique au boot

```bash
# scripts/start_lecture.sh
#!/bin/bash
# Démarrage du service de lecture au boot système

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
APP_DIR="$(dirname "$SCRIPT_DIR")"

cd "$APP_DIR"

# Activation de l'environnement virtuel
source venv/bin/activate

# Démarrage de l'application avec le service de lecture
export FLASK_ENV=production
export AUTO_START_LECTURE=true

python -c "
from app import create_app
app = create_app()
with app.app_context():
    from app.services import service_lecture
    if service_lecture:
        print('Démarrage du service de lecture...')
        service_lecture.demarrer()
        print('Service démarré. Appuyez sur Ctrl+C pour arrêter.')
        import time
        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            print('Arrêt du service...')
            service_lecture.arreter()
            print('Service arrêté.')
    else:
        print('Service de lecture non configuré.')
"
```

> **💡 À retenir** : La lecture continue transforme votre compteur en capteur permanent, fournissant des données temps réel pour analyse et automatisation.

> **⚠️ Astuce** : Commencez par des tests manuels avant d'activer la lecture automatique, et surveillez les logs pour détecter les problèmes précocement.

Dans le prochain chapitre, nous verrons comment envoyer ces données vers Home Assistant via MQTT pour une intégration domotique complète !

---

**Navigation**
- [Chapitre précédent : Authentification et sécurité](../Partie_III_Flask_Application/Chapitre_13_Authentification_Securite.md)
- [Chapitre suivant : Envoi vers Home Assistant (MQTT)](Chapitre_15_Envoi_Home_Assistant_MQTT.md)
- [Retour à la table des matières](../../README.md)
