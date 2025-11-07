# 🛠️ Chapitre 33 : Scripts utilitaires

## 🔧 Scripts d'installation et configuration

### Script d'installation automatique

```bash
#!/bin/bash
# install.sh - Installation automatique de l'application E450

set -e  # Arrêt en cas d'erreur

echo "=== Installation de l'application Landis+Gyr E450 ==="

# Vérification du système
if [[ "$OSTYPE" != "linux-gnu"* ]]; then
    echo "❌ Ce script est conçu pour Linux"
    exit 1
fi

# Vérification des droits root pour l'installation système
if [[ $EUID -eq 0 ]]; then
   echo "❌ Ne pas exécuter en root"
   exit 1
fi

# Création de l'environnement virtuel
echo "🐍 Création de l'environnement virtuel..."
python3 -m venv venv
source venv/bin/activate

# Mise à jour pip
pip install --upgrade pip

# Installation des dépendances
echo "📦 Installation des dépendances..."
pip install -r requirements.txt

# Configuration de la base de données
echo "🗄️ Initialisation de la base de données..."
flask db init
flask db migrate -m "Initialisation"
flask db upgrade

# Création de l'utilisateur admin
echo "👤 Création de l'utilisateur administrateur..."
python3 -c "
from app import create_app, db
from app.models import User

app = create_app()
with app.app_context():
    admin = User(username='admin', email='admin@localhost')
    admin.set_password('admin')
    admin.is_admin = True
    db.session.add(admin)
    db.session.commit()
    print('✅ Utilisateur admin créé (admin/admin)')
"

# Configuration des services système (optionnel)
read -p "⚙️ Configurer les services système (recommandé) ? (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "🔧 Configuration des services..."
    sudo cp deploy/e450-app.service /etc/systemd/system/
    sudo cp deploy/e450-collector.service /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable e450-app
    sudo systemctl enable e450-collector
fi

# Configuration des droits USB (pour port optique)
echo "🔌 Configuration des droits USB..."
sudo usermod -a -G dialout $USER
sudo usermod -a -G plugdev $USER

# Installation des utilitaires système
echo "🛠️ Installation des utilitaires..."
sudo apt-get update
sudo apt-get install -y \
    python3-dev \
    build-essential \
    libffi-dev \
    libssl-dev \
    python3-pip \
    sqlite3 \
    mosquitto \
    mosquitto-clients

echo "✅ Installation terminée !"
echo ""
echo "🚀 Pour démarrer l'application:"
echo "   source venv/bin/activate"
echo "   flask run"
echo ""
echo "📱 Interface accessible sur: http://localhost:5000"
echo "👤 Identifiants admin: admin / admin"
```

### Script de sauvegarde automatique

```bash
#!/bin/bash
# backup.sh - Sauvegarde automatique des données

BACKUP_DIR="/var/backups/e450"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/e450_backup_$DATE.tar.gz"

# Création du répertoire de sauvegarde
mkdir -p "$BACKUP_DIR"

echo "📦 Création de la sauvegarde: $BACKUP_FILE"

# Sauvegarde de la base de données
echo "🗄️ Sauvegarde de la base de données..."
cp instance/app.db "$BACKUP_DIR/app.db.$DATE"

# Sauvegarde des fichiers de configuration
echo "⚙️ Sauvegarde des configurations..."
tar -czf "$BACKUP_FILE" \
    --exclude='*.log' \
    --exclude='__pycache__' \
    --exclude='*.pyc' \
    -C /opt/e450 \
    .

# Nettoyage des anciennes sauvegardes (garde 7 jours)
echo "🧹 Nettoyage des anciennes sauvegardes..."
find "$BACKUP_DIR" -name "e450_backup_*.tar.gz" -mtime +7 -delete
find "$BACKUP_DIR" -name "app.db.*" -mtime +7 -delete

# Calcul de la taille
SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
echo "✅ Sauvegarde créée: $SIZE"

# Test d'intégrité
echo "🔍 Vérification de l'intégrité..."
if tar -tzf "$BACKUP_FILE" > /dev/null 2>&1; then
    echo "✅ Archive valide"
else
    echo "❌ Archive corrompue !"
    exit 1
fi
```

## 📊 Scripts de monitoring et diagnostic

### Moniteur de santé du système

```python
#!/usr/bin/env python3
# health_monitor.py - Moniteur de santé du système E450

import psutil
import time
import logging
from datetime import datetime, timedelta
from app import create_app
from app.models import MesureEnergie
from app.services.lecteur_compteur import LecteurCompteurE450

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class HealthMonitor:
    """Moniteur de santé du système"""

    def __init__(self):
        self.app = create_app()
        self.dernier_check = datetime.utcnow()
        self.intervalle_check = timedelta(minutes=5)

    def check_sante_generale(self):
        """Vérification générale de santé"""
        with self.app.app_context():
            sante = {
                'timestamp': datetime.utcnow().isoformat(),
                'systeme': self._check_systeme(),
                'application': self._check_application(),
                'base_donnees': self._check_base_donnees(),
                'compteurs': self._check_compteurs(),
                'services_externes': self._check_services_externes()
            }

            # Score global
            scores = [v.get('score', 0) for v in sante.values() if isinstance(v, dict)]
            sante['score_global'] = sum(scores) / len(scores) if scores else 0

            return sante

    def _check_systeme(self):
        """Vérification du système hôte"""
        try:
            # CPU
            cpu_percent = psutil.cpu_percent(interval=1)

            # Mémoire
            mem = psutil.virtual_memory()
            mem_percent = mem.percent

            # Disque
            disk = psutil.disk_usage('/')
            disk_percent = disk.percent

            # Réseau
            net = psutil.net_io_counters()
            bytes_sent = net.bytes_sent
            bytes_recv = net.bytes_recv

            # Score basé sur les seuils
            score = 100
            if cpu_percent > 80: score -= 20
            if mem_percent > 85: score -= 20
            if disk_percent > 90: score -= 20

            return {
                'cpu_percent': cpu_percent,
                'mem_percent': mem_percent,
                'disk_percent': disk_percent,
                'network': {
                    'bytes_sent': bytes_sent,
                    'bytes_recv': bytes_recv
                },
                'score': max(0, score),
                'status': 'healthy' if score >= 80 else 'warning' if score >= 60 else 'critical'
            }

        except Exception as e:
            logger.error(f"Erreur check système: {e}")
            return {'error': str(e), 'score': 0}

    def _check_application(self):
        """Vérification de l'application Flask"""
        try:
            # Vérifier si le processus tourne
            app_running = False
            for proc in psutil.process_iter(['pid', 'name', 'cmdline']):
                if 'python' in proc.info['name'].lower():
                    cmdline = ' '.join(proc.info['cmdline'])
                    if 'run.py' in cmdline or 'flask' in cmdline:
                        app_running = True
                        break

            # Utilisation mémoire de l'app
            mem_usage = 0
            if app_running:
                # Simulation - en production utiliser des métriques réelles
                mem_usage = 50  # Mo

            score = 100 if app_running else 0

            return {
                'running': app_running,
                'memory_usage': mem_usage,
                'uptime': 'N/A',  # À implémenter avec psutil
                'score': score,
                'status': 'healthy' if app_running else 'critical'
            }

        except Exception as e:
            logger.error(f"Erreur check application: {e}")
            return {'error': str(e), 'score': 0}

    def _check_base_donnees(self):
        """Vérification de la base de données"""
        try:
            from app import db

            # Test de connexion
            db.engine.execute('SELECT 1')

            # Statistiques
            mesures_count = MesureEnergie.query.count()
            dernier_mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

            # Taille de la base
            import os
            db_path = self.app.config['SQLALCHEMY_DATABASE_URI'].replace('sqlite:///', '')
            if os.path.exists(db_path):
                db_size = os.path.getsize(db_path) / (1024 * 1024)  # Mo
            else:
                db_size = 0

            # Vérifier fraîcheur des données
            if dernier_mesure:
                age_dernier = datetime.utcnow() - dernier_mesure.timestamp
                donnees_fraiches = age_dernier < timedelta(hours=1)
            else:
                donnees_fraiches = False

            score = 100
            if not donnees_fraiches: score -= 30
            if db_size > 100: score -= 10  # Avertissement taille

            return {
                'connection': True,
                'mesures_count': mesures_count,
                'db_size_mb': db_size,
                'dernier_mesure': dernier_mesure.timestamp.isoformat() if dernier_mesure else None,
                'donnees_fraiches': donnees_fraiches,
                'score': score,
                'status': 'healthy' if score >= 80 else 'warning'
            }

        except Exception as e:
            logger.error(f"Erreur check base de données: {e}")
            return {'error': str(e), 'score': 0}

    def _check_compteurs(self):
        """Vérification des compteurs configurés"""
        try:
            from app.models import ConfigurationCompteur

            compteurs = ConfigurationCompteur.query.all()
            resultats = []

            for compteur in compteurs:
                if compteur.actif:
                    # Test de connexion
                    config_lecteur = {
                        'interface': compteur.type_interface,
                        'port': '/dev/ttyUSB0',  # À adapter
                        'compteur_id': compteur.compteur_id
                    }

                    lecteur = LecteurCompteurE450(config_lecteur)
                    connexion_ok = lecteur.connexion_compteur()

                    # Dernière mesure
                    dernier = MesureEnergie.query.filter_by(
                        compteur_id=compteur.compteur_id
                    ).order_by(MesureEnergie.timestamp.desc()).first()

                    age_dernier = datetime.utcnow() - dernier.timestamp if dernier else timedelta(days=1)

                    resultats.append({
                        'id': compteur.compteur_id,
                        'nom': compteur.nom_affiche,
                        'connexion_ok': connexion_ok,
                        'dernier_mesure': dernier.timestamp.isoformat() if dernier else None,
                        'age_dernier_heures': age_dernier.total_seconds() / 3600
                    })

            # Score global
            connexions_ok = sum(1 for r in resultats if r['connexion_ok'])
            score = (connexions_ok / len(resultats)) * 100 if resultats else 100

            return {
                'compteurs_actifs': len(resultats),
                'connexions_ok': connexions_ok,
                'detail': resultats,
                'score': score,
                'status': 'healthy' if score >= 80 else 'warning' if score >= 50 else 'critical'
            }

        except Exception as e:
            logger.error(f"Erreur check compteurs: {e}")
            return {'error': str(e), 'score': 0}

    def _check_services_externes(self):
        """Vérification des services externes (MQTT, InfluxDB)"""
        try:
            services = {}

            # MQTT
            try:
                import paho.mqtt.client as mqtt
                client = mqtt.Client()
                if self.app.config.get('MQTT_USERNAME'):
                    client.username_pw_set(
                        self.app.config['MQTT_USERNAME'],
                        self.app.config['MQTT_PASSWORD']
                    )
                client.connect(self.app.config.get('MQTT_BROKER', 'localhost'),
                             self.app.config.get('MQTT_PORT', 1883), 5)
                client.disconnect()
                services['mqtt'] = {'status': 'ok', 'message': 'Connexion réussie'}
            except Exception as e:
                services['mqtt'] = {'status': 'error', 'message': str(e)}

            # InfluxDB (si configuré)
            if self.app.config.get('INFLUXDB_URL'):
                try:
                    from influxdb_client import InfluxDBClient
                    client = InfluxDBClient(
                        url=self.app.config['INFLUXDB_URL'],
                        token=self.app.config['INFLUXDB_TOKEN'],
                        org=self.app.config['INFLUXDB_ORG']
                    )
                    health = client.health()
                    services['influxdb'] = {'status': 'ok', 'message': 'Connexion réussie'}
                except Exception as e:
                    services['influxdb'] = {'status': 'error', 'message': str(e)}
            else:
                services['influxdb'] = {'status': 'not_configured'}

            # Score
            services_ok = sum(1 for s in services.values() if s['status'] == 'ok')
            total_services = len([s for s in services.values() if s['status'] != 'not_configured'])
            score = (services_ok / total_services) * 100 if total_services > 0 else 100

            return {
                'services': services,
                'score': score,
                'status': 'healthy' if score >= 80 else 'warning'
            }

        except Exception as e:
            logger.error(f"Erreur check services externes: {e}")
            return {'error': str(e), 'score': 0}

def main():
    """Point d'entrée principal"""
    monitor = HealthMonitor()

    while True:
        try:
            sante = monitor.check_sante_generale()

            # Log du résultat
            score_global = sante.get('score_global', 0)
            status = '🟢' if score_global >= 80 else '🟡' if score_global >= 60 else '🔴'

            logger.info(f"{status} Score santé système: {score_global:.1f}%")

            # Affichage détaillé si warning/critical
            if score_global < 80:
                for categorie, details in sante.items():
                    if isinstance(details, dict) and details.get('score', 100) < 80:
                        logger.warning(f"⚠️ {categorie}: {details}")

        except Exception as e:
            logger.error(f"Erreur monitoring: {e}")

        time.sleep(300)  # Vérification toutes les 5 minutes

if __name__ == '__main__':
    main()
```

### Script de diagnostic des communications

```python
#!/usr/bin/env python3
# diagnostic_com.py - Diagnostic des communications compteur

import serial
import time
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class DiagnosticCommunications:
    """Outil de diagnostic des communications"""

    def __init__(self):
        self.resultats = []

    def diagnostiquer_port_optique(self, port='/dev/ttyUSB0'):
        """Diagnostic complet du port optique"""
        logger.info(f"🔍 Diagnostic port optique: {port}")

        config = {
            'baudrate': 300,
            'bytesize': serial.SEVENBITS,
            'parity': serial.PARITY_EVEN,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 5
        }

        try:
            with serial.Serial(port, **config) as ser:
                logger.info("✅ Port série ouvert")

                # Test 1: Identification compteur
                logger.info("Test 1: Identification compteur (mode A)")
                ser.write(b'/?!\r\n')
                time.sleep(0.5)
                reponse = ser.read(100)

                if reponse:
                    logger.info(f"✅ Réponse reçue: {reponse}")
                    self.resultats.append({
                        'test': 'identification',
                        'resultat': 'succes',
                        'reponse': reponse.decode('ascii', errors='ignore').strip()
                    })
                else:
                    logger.error("❌ Aucune réponse")
                    self.resultats.append({
                        'test': 'identification',
                        'resultat': 'echec',
                        'erreur': 'Aucune réponse'
                    })
                    return False

                # Test 2: Passage en mode C
                logger.info("Test 2: Passage en mode C")
                ser.write(b'\x06000\r\n')
                time.sleep(0.5)
                reponse = ser.read(50)

                if reponse:
                    logger.info(f"✅ Mode C activé: {reponse}")
                    self.resultats.append({
                        'test': 'mode_c',
                        'resultat': 'succes',
                        'reponse': reponse.decode('ascii', errors='ignore').strip()
                    })
                else:
                    logger.warning("⚠️ Pas de réponse mode C (normal pour certains compteurs)")

                # Test 3: Lecture d'un code simple
                logger.info("Test 3: Lecture code OBIS 0.0.0 (numéro de série)")
                ser.write(b'R1\r\n0.0.0()\r\n')
                time.sleep(0.5)
                reponse = ser.read(200)

                if reponse:
                    logger.info(f"✅ Données lues: {reponse}")
                    self.resultats.append({
                        'test': 'lecture_obis',
                        'resultat': 'succes',
                        'reponse': reponse.decode('ascii', errors='ignore').strip()
                    })
                else:
                    logger.error("❌ Impossible de lire les données")
                    self.resultats.append({
                        'test': 'lecture_obis',
                        'resultat': 'echec',
                        'erreur': 'Pas de réponse'
                    })

                return True

        except serial.SerialException as e:
            logger.error(f"❌ Erreur port série: {e}")
            self.resultats.append({
                'test': 'connexion_serie',
                'resultat': 'echec',
                'erreur': str(e)
            })
            return False

    def diagnostiquer_mbus(self, port='/dev/ttyUSB1', adresse=1):
        """Diagnostic du bus M-Bus"""
        logger.info(f"🔍 Diagnostic M-Bus: {port} (adresse {adresse})")

        config = {
            'baudrate': 9600,
            'bytesize': serial.EIGHTBITS,
            'parity': serial.PARITY_NONE,
            'stopbits': serial.STOPBITS_ONE,
            'timeout': 2
        }

        try:
            with serial.Serial(port, **config) as ser:
                logger.info("✅ Port M-Bus ouvert")

                # Test 1: Ping M-Bus
                logger.info("Test 1: Ping équipement")
                requete_ping = bytes([0x68, 0x03, 0x03, 0x68, adresse, 0x00, 0x00])

                ser.write(requete_ping)
                time.sleep(1)
                reponse = ser.read(20)

                if reponse:
                    logger.info(f"✅ Équipement répond: {reponse.hex()}")
                    self.resultats.append({
                        'test': 'ping_mbus',
                        'resultat': 'succes',
                        'reponse': reponse.hex()
                    })
                else:
                    logger.error("❌ Équipement ne répond pas")
                    self.resultats.append({
                        'test': 'ping_mbus',
                        'resultat': 'echec',
                        'erreur': 'Pas de réponse'
                    })
                    return False

                return True

        except serial.SerialException as e:
            logger.error(f"❌ Erreur port M-Bus: {e}")
            self.resultats.append({
                'test': 'connexion_mbus',
                'resultat': 'echec',
                'erreur': str(e)
            })
            return False

    def generer_rapport(self):
        """Génère un rapport de diagnostic"""
        rapport = {
            'date': datetime.utcnow().isoformat(),
            'tests_executés': len(self.resultats),
            'tests_reussis': len([r for r in self.resultats if r['resultat'] == 'succes']),
            'tests_echoués': len([r for r in self.resultats if r['resultat'] == 'echec']),
            'detail_tests': self.resultats
        }

        # Diagnostic global
        taux_succes = rapport['tests_reussis'] / rapport['tests_executés'] if rapport['tests_executés'] > 0 else 0

        if taux_succes == 1.0:
            rapport['diagnostic_global'] = 'excellent'
            rapport['recommandations'] = ['Communications opérationnelles']
        elif taux_succes >= 0.7:
            rapport['diagnostic_global'] = 'bon'
            rapport['recommandations'] = ['Communications stables avec quelques points d\'amélioration']
        elif taux_succes >= 0.5:
            rapport['diagnostic_global'] = 'moyen'
            rapport['recommandations'] = ['Problèmes de communication à résoudre']
        else:
            rapport['diagnostic_global'] = 'critique'
            rapport['recommandations'] = ['Communications défaillantes - intervention requise']

        return rapport

    def afficher_rapport(self, rapport):
        """Affiche le rapport de façon lisible"""
        print("\n" + "="*50)
        print("📋 RAPPORT DE DIAGNOSTIC COMMUNICATIONS")
        print("="*50)
        print(f"Date: {rapport['date']}")
        print(f"Tests exécutés: {rapport['tests_executés']}")
        print(f"Tests réussis: {rapport['tests_reussis']}")
        print(f"Tests échoués: {rapport['tests_echoués']}")
        print(f"Diagnostic global: {rapport['diagnostic_global'].upper()}")
        print()

        print("🔍 DÉTAIL DES TESTS:")
        for test in rapport['detail_tests']:
            status = "✅" if test['resultat'] == 'succes' else "❌"
            print(f"  {status} {test['test']}: {test['resultat']}")
            if 'reponse' in test:
                print(f"      Réponse: {test['reponse']}")
            if 'erreur' in test:
                print(f"      Erreur: {test['erreur']}")

        print()
        print("💡 RECOMMANDATIONS:")
        for rec in rapport['recommandations']:
            print(f"  • {rec}")

        print("="*50)

def main():
    """Point d'entrée principal"""
    diag = DiagnosticCommunications()

    print("🔧 Outil de diagnostic des communications E450")
    print("Sélectionnez l'interface à tester:")
    print("1. Port optique")
    print("2. Bus M-Bus")
    print("3. Les deux")

    choix = input("Votre choix (1-3): ").strip()

    if choix in ['1', '3']:
        print("\n🔌 Test du port optique...")
        diag.diagnostiquer_port_optique()

    if choix in ['2', '3']:
        print("\n🔗 Test du bus M-Bus...")
        adresse = input("Adresse M-Bus (défaut 1): ").strip()
        adresse = int(adresse) if adresse.isdigit() else 1
        diag.diagnostiquer_mbus(adresse=adresse)

    # Génération du rapport
    rapport = diag.generer_rapport()
    diag.afficher_rapport(rapport)

if __name__ == '__main__':
    main()
```

---

**Navigation**
- [Chapitre précédent : Code source application](Chapitre_32_Code_Source_Application.md)
- [Chapitre suivant : API REST complète](Chapitre_34_API_REST_Complete.md)
- [Retour à la table des matières](../../README.md)
