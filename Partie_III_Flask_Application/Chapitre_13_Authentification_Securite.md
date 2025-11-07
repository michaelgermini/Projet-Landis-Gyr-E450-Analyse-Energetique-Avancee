# 💻 Chapitre 13 : Authentification et sécurité

## 🔐 Système d'authentification

Notre application implémente un système d'authentification robuste basé sur Flask-Login, avec gestion des sessions sécurisées et protection CSRF.

### Modèle utilisateur

```python
# app/models.py (extension)
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
from flask import current_app

class User(UserMixin, db.Model):
    """Modèle utilisateur avec authentification avancée"""
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False, index=True)
    email = db.Column(db.String(120), unique=True, nullable=False, index=True)
    password_hash = db.Column(db.String(128), nullable=False)
    is_admin = db.Column(db.Boolean, default=False)
    is_active = db.Column(db.Boolean, default=True)

    # Sécurité
    failed_login_attempts = db.Column(db.Integer, default=0)
    locked_until = db.Column(db.DateTime, nullable=True)
    last_login = db.Column(db.DateTime)
    password_changed_at = db.Column(db.DateTime, default=datetime.utcnow)

    # Profil
    first_name = db.Column(db.String(64))
    last_name = db.Column(db.String(64))
    phone = db.Column(db.String(20))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relations
    mesures = db.relationship('MesureEnergie', backref='user', lazy='dynamic')
    parametres = db.relationship('UserPreference', backref='user', lazy='dynamic')

    def set_password(self, password):
        """Hash et définit le mot de passe"""
        if not self._validate_password_strength(password):
            raise ValueError("Mot de passe trop faible")

        self.password_hash = generate_password_hash(password)
        self.password_changed_at = datetime.utcnow()
        self.failed_login_attempts = 0  # Reset des tentatives
        self.locked_until = None  # Déverrouillage

    def check_password(self, password):
        """Vérifie le mot de passe"""
        # Vérification du verrouillage
        if self.locked_until and datetime.utcnow() < self.locked_until:
            return False

        # Vérification du mot de passe
        if check_password_hash(self.password_hash, password):
            self.failed_login_attempts = 0
            self.last_login = datetime.utcnow()
            db.session.commit()
            return True
        else:
            self.failed_login_attempts += 1

            # Verrouillage après 5 tentatives
            if self.failed_login_attempts >= 5:
                self.locked_until = datetime.utcnow() + timedelta(minutes=15)

            db.session.commit()
            return False

    def _validate_password_strength(self, password):
        """Validation de la force du mot de passe"""
        if len(password) < 8:
            return False
        if not re.search(r'[A-Z]', password):
            return False
        if not re.search(r'[a-z]', password):
            return False
        if not re.search(r'\d', password):
            return False
        return True

    def get_reset_token(self, expires_sec=1800):
        """Génère un token de reset de mot de passe"""
        s = Serializer(current_app.config['SECRET_KEY'], expires_sec)
        return s.dumps({'user_id': self.id}).decode('utf-8')

    @staticmethod
    def verify_reset_token(token):
        """Vérifie un token de reset"""
        s = Serializer(current_app.config['SECRET_KEY'])
        try:
            user_id = s.loads(token)['user_id']
        except:
            return None
        return User.query.get(user_id)

    def is_locked(self):
        """Vérifie si le compte est verrouillé"""
        return self.locked_until and datetime.utcnow() < self.locked_until

    def get_lock_time_remaining(self):
        """Temps restant avant déverrouillage (en minutes)"""
        if not self.is_locked():
            return 0
        return int((self.locked_until - datetime.utcnow()).total_seconds() / 60)

    def __repr__(self):
        return f'<User {self.username}>'

class UserPreference(db.Model):
    """Préférences utilisateur"""
    __tablename__ = 'user_preferences'

    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    preference_key = db.Column(db.String(64), nullable=False)
    preference_value = db.Column(db.String(255))

    __table_args__ = (
        db.UniqueConstraint('user_id', 'preference_key', name='unique_user_preference'),
    )

    def __repr__(self):
        return f'<UserPreference {self.preference_key}={self.preference_value}>'
```

### Routes d'authentification

```python
# app/routes/auth.py
from flask import Blueprint, render_template, redirect, url_for, flash, request
from flask_login import login_user, logout_user, login_required, current_user
from werkzeug.urls import url_parse
from app.models import User, db
from app.forms import LoginForm, RegistrationForm, ResetPasswordForm, ChangePasswordForm
import re

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    """Page de connexion"""
    if current_user.is_authenticated:
        return redirect(url_for('main.dashboard'))

    form = LoginForm()

    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()

        if user and user.check_password(form.password.data) and user.is_active:
            if user.is_locked():
                remaining = user.get_lock_time_remaining()
                flash(f'Compte verrouillé. Réessayez dans {remaining} minutes.', 'error')
                return redirect(url_for('auth.login'))

            login_user(user, remember=form.remember_me.data)

            # Log de connexion réussie
            current_app.logger.info(f"Connexion réussie: {user.username}")

            next_page = request.args.get('next')
            if not next_page or url_parse(next_page).netloc != '':
                next_page = url_for('main.dashboard')
            return redirect(next_page)

        else:
            flash('Nom d\'utilisateur ou mot de passe incorrect.', 'error')
            current_app.logger.warning(f"Tentative de connexion échouée: {form.username.data}")

    return render_template('auth/login.html', form=form)

@auth_bp.route('/logout')
@login_required
def logout():
    """Déconnexion"""
    current_app.logger.info(f"Déconnexion: {current_user.username}")
    logout_user()
    return redirect(url_for('main.index'))

@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    """Inscription d'un nouvel utilisateur"""
    if current_user.is_authenticated:
        return redirect(url_for('main.dashboard'))

    form = RegistrationForm()

    if form.validate_on_submit():
        # Validation supplémentaire
        if User.query.filter_by(username=form.username.data).first():
            flash('Ce nom d\'utilisateur est déjà pris.', 'error')
            return render_template('auth/register.html', form=form)

        if User.query.filter_by(email=form.email.data).first():
            flash('Cette adresse email est déjà utilisée.', 'error')
            return render_template('auth/register.html', form=form)

        # Création de l'utilisateur
        user = User(
            username=form.username.data,
            email=form.email.data,
            first_name=form.first_name.data,
            last_name=form.last_name.data
        )
        user.set_password(form.password.data)

        db.session.add(user)
        db.session.commit()

        # Création des préférences par défaut
        default_prefs = {
            'theme': 'neon',
            'notifications_email': 'true',
            'dashboard_refresh': '30'
        }

        for key, value in default_prefs.items():
            pref = UserPreference(user_id=user.id, preference_key=key, preference_value=value)
            db.session.add(pref)

        db.session.commit()

        flash('Inscription réussie ! Vous pouvez maintenant vous connecter.', 'success')
        current_app.logger.info(f"Nouvel utilisateur inscrit: {user.username}")

        return redirect(url_for('auth.login'))

    return render_template('auth/register.html', form=form)

@auth_bp.route('/reset-password', methods=['GET', 'POST'])
def reset_password_request():
    """Demande de reset de mot de passe"""
    if current_user.is_authenticated:
        return redirect(url_for('main.dashboard'))

    form = ResetPasswordForm()

    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()

        if user:
            token = user.get_reset_token()
            # Envoi d'email (implémentation dans un chapitre ultérieur)
            send_password_reset_email(user, token)

        flash('Si cette adresse email existe, vous recevrez un lien de réinitialisation.', 'info')
        return redirect(url_for('auth.login'))

    return render_template('auth/reset_password_request.html', form=form)

@auth_bp.route('/reset-password/<token>', methods=['GET', 'POST'])
def reset_password(token):
    """Reset du mot de passe avec token"""
    if current_user.is_authenticated:
        return redirect(url_for('main.dashboard'))

    user = User.verify_reset_token(token)

    if not user:
        flash('Token invalide ou expiré.', 'error')
        return redirect(url_for('auth.reset_password_request'))

    form = ChangePasswordForm()

    if form.validate_on_submit():
        user.set_password(form.password.data)
        db.session.commit()

        flash('Votre mot de passe a été réinitialisé.', 'success')
        current_app.logger.info(f"Reset de mot de passe: {user.username}")

        return redirect(url_for('auth.login'))

    return render_template('auth/reset_password.html', form=form)

@auth_bp.route('/change-password', methods=['GET', 'POST'])
@login_required
def change_password():
    """Changement de mot de passe (utilisateur connecté)"""
    form = ChangePasswordForm()

    if form.validate_on_submit():
        if not current_user.check_password(form.old_password.data):
            flash('Ancien mot de passe incorrect.', 'error')
            return render_template('auth/change_password.html', form=form)

        current_user.set_password(form.password.data)
        db.session.commit()

        flash('Mot de passe changé avec succès.', 'success')
        current_app.logger.info(f"Changement de mot de passe: {current_user.username}")

        return redirect(url_for('main.dashboard'))

    return render_template('auth/change_password.html', form=form)
```

### Formulaires avec WTForms

```python
# app/forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField, ValidationError
from wtforms.validators import DataRequired, Email, EqualTo, Length, Regexp
import re

class LoginForm(FlaskForm):
    """Formulaire de connexion"""
    username = StringField('Nom d\'utilisateur',
                          validators=[DataRequired(message="Nom d'utilisateur requis")],
                          render_kw={"placeholder": "Votre nom d'utilisateur"})
    password = PasswordField('Mot de passe',
                            validators=[DataRequired(message="Mot de passe requis")],
                            render_kw={"placeholder": "Votre mot de passe"})
    remember_me = BooleanField('Se souvenir de moi')
    submit = SubmitField('Se connecter')

class RegistrationForm(FlaskForm):
    """Formulaire d'inscription"""
    username = StringField('Nom d\'utilisateur',
                          validators=[
                              DataRequired(),
                              Length(min=3, max=64),
                              Regexp(r'^[a-zA-Z0-9_]+$', message="Caractères alphanumériques uniquement")
                          ])
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    first_name = StringField('Prénom', validators=[Length(max=64)])
    last_name = StringField('Nom', validators=[Length(max=64)])
    password = PasswordField('Mot de passe',
                            validators=[
                                DataRequired(),
                                Length(min=8),
                                Regexp(r'(?=.*[a-z])(?=.*[A-Z])(?=.*\d)',
                                      message="Le mot de passe doit contenir au moins une minuscule, une majuscule et un chiffre")
                            ])
    password2 = PasswordField('Répéter le mot de passe',
                             validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('S\'inscrire')

    def validate_username(self, username):
        """Validation personnalisée du nom d'utilisateur"""
        if len(username.data) < 3:
            raise ValidationError('Le nom d\'utilisateur doit faire au moins 3 caractères.')

    def validate_email(self, email):
        """Validation personnalisée de l'email"""
        # Vérification du domaine
        domain = email.data.split('@')[-1]
        blocked_domains = ['10minutemail.com', 'guerrillamail.com']  # Exemple

        if domain in blocked_domains:
            raise ValidationError('Domaine d\'email non autorisé.')

class ResetPasswordForm(FlaskForm):
    """Formulaire de demande de reset"""
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    submit = SubmitField('Envoyer le lien de réinitialisation')

class ChangePasswordForm(FlaskForm):
    """Formulaire de changement de mot de passe"""
    old_password = PasswordField('Ancien mot de passe',
                                validators=[DataRequired()])
    password = PasswordField('Nouveau mot de passe',
                            validators=[
                                DataRequired(),
                                Length(min=8),
                                Regexp(r'(?=.*[a-z])(?=.*[A-Z])(?=.*\d)')
                            ])
    password2 = PasswordField('Répéter le nouveau mot de passe',
                             validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Changer le mot de passe')
```

## 🛡️ Mesures de sécurité

### Protection CSRF

```python
# app/__init__.py (extension)
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect()

def create_app(config_name=None):
    # ...
    csrf.init_app(app)
    # ...
```

### Headers de sécurité

```python
# app/__init__.py (extension)
@app.after_request
def add_security_headers(response):
    """Ajout d'headers de sécurité"""
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'SAMEORIGIN'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    response.headers['Content-Security-Policy'] = (
        "default-src 'self'; "
        "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
        "font-src 'self' https://fonts.gstatic.com; "
        "img-src 'self' data: https:; "
        "connect-src 'self' ws: wss:"
    )
    return response
```

### Rate limiting

```python
# app/extensions.py (extension)
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

# Dans app/__init__.py
limiter.init_app(app)

# Application aux routes sensibles
@auth_bp.route('/login', methods=['POST'])
@limiter.limit("5 per minute", methods=["POST"])
def login():
    # ...
```

### Sanitisation des entrées

```python
# app/utils/security.py
import bleach
from html import escape

def sanitize_html(text):
    """Nettoie le HTML dangereux"""
    allowed_tags = ['p', 'br', 'strong', 'em', 'u']
    allowed_attrs = {}

    return bleach.clean(text, tags=allowed_tags, attributes=allowed_attrs, strip=True)

def sanitize_filename(filename):
    """Nettoie un nom de fichier"""
    import re
    # Suppression des caractères dangereux
    filename = re.sub(r'[<>:"/\\|?*]', '', filename)
    # Limitation de longueur
    return filename[:255]

def validate_input(data, rules):
    """Validation générique des entrées"""
    errors = []

    for field, rule in rules.items():
        value = data.get(field)

        if rule.get('required') and not value:
            errors.append(f"{field} est requis")
            continue

        if 'min_length' in rule and len(str(value)) < rule['min_length']:
            errors.append(f"{field} doit faire au moins {rule['min_length']} caractères")

        if 'max_length' in rule and len(str(value)) > rule['max_length']:
            errors.append(f"{field} ne peut pas dépasser {rule['max_length']} caractères")

        if 'pattern' in rule and not re.match(rule['pattern'], str(value)):
            errors.append(f"{field} ne respecte pas le format attendu")

    return errors
```

## 🔑 Gestion des sessions

### Configuration avancée

```python
# config.py (extension)
class Config:
    # ...

    # Sessions
    SECRET_KEY = os.environ.get('SECRET_KEY') or os.urandom(32)
    SESSION_TYPE = 'filesystem'  # Ou 'redis' en production
    SESSION_FILE_DIR = '/tmp/flask_sessions'
    SESSION_PERMANENT = True
    PERMANENT_SESSION_LIFETIME = timedelta(days=7)

    # Sécurité des sessions
    SESSION_COOKIE_SECURE = True  # HTTPS uniquement
    SESSION_COOKIE_HTTPONLY = True  # JavaScript ne peut pas accéder
    SESSION_COOKIE_SAMESITE = 'Lax'  # Protection CSRF
```

### Gestion des sessions utilisateurs

```python
# app/routes/auth.py (extension)
@auth_bp.route('/profile')
@login_required
def profile():
    """Profil utilisateur"""
    return render_template('auth/profile.html', user=current_user)

@auth_bp.route('/preferences', methods=['GET', 'POST'])
@login_required
def preferences():
    """Gestion des préférences utilisateur"""

    if request.method == 'POST':
        # Mise à jour des préférences
        theme = request.form.get('theme', 'neon')
        notifications = request.form.get('notifications', 'false') == 'true'
        refresh_rate = request.form.get('refresh_rate', '30')

        # Sauvegarde en base
        UserPreference.update_or_create(
            user_id=current_user.id,
            preference_key='theme',
            preference_value=theme
        )
        # ... autres préférences

        flash('Préférences sauvegardées.', 'success')
        return redirect(url_for('auth.preferences'))

    # Récupération des préférences actuelles
    prefs = {}
    for pref in current_user.parametres:
        prefs[pref.preference_key] = pref.preference_value

    return render_template('auth/preferences.html', preferences=prefs)
```

## 📧 Système de notifications

### Notifications par email

```python
# app/services/notifications.py
from flask import render_template
from flask_mail import Mail, Message
from app import create_app

class NotificationService:
    """Service de notifications"""

    def __init__(self, app=None):
        self.mail = Mail(app) if app else None

    def init_app(self, app):
        self.mail = Mail(app)

    def send_email(self, subject, recipients, template, **kwargs):
        """Envoi d'email générique"""
        if not self.mail:
            return False

        try:
            msg = Message(
                subject=subject,
                recipients=recipients,
                sender=app.config['MAIL_DEFAULT_SENDER']
            )

            # Rendu du template HTML
            msg.html = render_template(f'emails/{template}.html', **kwargs)

            # Version texte
            msg.body = render_template(f'emails/{template}.txt', **kwargs)

            self.mail.send(msg)
            return True

        except Exception as e:
            app.logger.error(f"Erreur envoi email: {e}")
            return False

    def notify_login(self, user, ip_address):
        """Notification de connexion"""
        if user.email and self.user_wants_notifications(user, 'login'):
            self.send_email(
                subject="Nouvelle connexion à votre compte",
                recipients=[user.email],
                template='login_notification',
                user=user,
                ip_address=ip_address,
                timestamp=datetime.utcnow()
            )

    def notify_anomaly(self, user, anomaly):
        """Notification d'anomalie"""
        if user.email and self.user_wants_notifications(user, 'anomalies'):
            self.send_email(
                subject=f"Alerte: {anomaly['titre']}",
                recipients=[user.email],
                template='anomaly_alert',
                user=user,
                anomaly=anomaly
            )

    def user_wants_notifications(self, user, notification_type):
        """Vérifie si l'utilisateur veut ce type de notifications"""
        pref = UserPreference.query.filter_by(
            user_id=user.id,
            preference_key=f'notifications_{notification_type}'
        ).first()

        return pref and pref.preference_value.lower() == 'true'
```

### Notifications en temps réel

```python
# app/services/websocket_notifications.py
from flask_socketio import emit, join_room, leave_room

class WebSocketNotifications:
    """Notifications via WebSocket"""

    @staticmethod
    def notify_user(user_id, event_type, data):
        """Notification d'un utilisateur spécifique"""
        room = f'user_{user_id}'
        emit(event_type, data, room=room)

    @staticmethod
    def notify_all(event_type, data):
        """Notification de tous les utilisateurs connectés"""
        emit(event_type, data, broadcast=True)

    @staticmethod
    def on_connect():
        """Gestion de la connexion WebSocket"""
        if current_user.is_authenticated:
            join_room(f'user_{current_user.id}')
            join_room('general')

    @staticmethod
    def on_disconnect():
        """Gestion de la déconnexion"""
        if current_user.is_authenticated:
            leave_room(f'user_{current_user.id}')
            leave_room('general')

# Intégration SocketIO
from flask_socketio import SocketIO

socketio = SocketIO()

@socketio.on('connect')
def handle_connect():
    WebSocketNotifications.on_connect()

@socketio.on('disconnect')
def handle_disconnect():
    WebSocketNotifications.on_disconnect()
```

## 🛡️ Audit et logging

### Système de logs sécurisé

```python
# app/utils/audit.py
import logging
from logging.handlers import RotatingFileHandler
import os
from flask import request

class AuditLogger:
    """Logger d'audit pour les actions sensibles"""

    def __init__(self, app=None):
        self.app = app
        self.audit_logger = None

        if app:
            self.init_app(app)

    def init_app(self, app):
        """Initialisation du logger d'audit"""
        audit_log_path = os.path.join(app.root_path, '..', 'logs', 'audit.log')

        # Création du répertoire si nécessaire
        os.makedirs(os.path.dirname(audit_log_path), exist_ok=True)

        # Configuration du handler
        handler = RotatingFileHandler(
            audit_log_path,
            maxBytes=10*1024*1024,  # 10MB
            backupCount=5
        )

        # Format sécurisé
        formatter = logging.Formatter(
            '%(asctime)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)

        # Création du logger
        self.audit_logger = logging.getLogger('audit')
        self.audit_logger.setLevel(logging.INFO)
        self.audit_logger.addHandler(handler)

        # Injection dans l'app
        app.audit_logger = self.audit_logger

    def log_action(self, action, user=None, details=None, level='INFO'):
        """Log d'une action d'audit"""
        if not self.audit_logger:
            return

        # Collecte des informations de contexte
        ip = request.remote_addr if request else 'unknown'
        user_agent = request.headers.get('User-Agent', 'unknown') if request else 'unknown'

        # Construction du message
        message_parts = [
            f"ACTION={action}",
            f"IP={ip}",
            f"USER_AGENT={user_agent[:100]}"  # Limitation de longueur
        ]

        if user:
            message_parts.append(f"USER={user}")

        if details:
            # Sanitisation des détails sensibles
            safe_details = self._sanitize_details(details)
            message_parts.append(f"DETAILS={safe_details}")

        message = ' | '.join(message_parts)

        # Log selon le niveau
        if level == 'WARNING':
            self.audit_logger.warning(message)
        elif level == 'ERROR':
            self.audit_logger.error(message)
        else:
            self.audit_logger.info(message)

    def _sanitize_details(self, details):
        """Suppression des informations sensibles"""
        if isinstance(details, dict):
            # Suppression des mots de passe, tokens, etc.
            sanitized = details.copy()
            sensitive_keys = ['password', 'token', 'secret', 'key']

            for key in sensitive_keys:
                if key in sanitized:
                    sanitized[key] = '[REDACTED]'

            return str(sanitized)
        return str(details)

# Instance globale
audit_logger = AuditLogger()

# Utilisation dans les routes
@auth_bp.route('/login', methods=['POST'])
def login():
    # ...
    if user and user.check_password(form.password.data):
        audit_logger.log_action(
            'LOGIN_SUCCESS',
            user=user.username,
            details={'method': 'password'}
        )
        # ...
    else:
        audit_logger.log_action(
            'LOGIN_FAILED',
            user=form.username.data,
            details={'reason': 'invalid_credentials'},
            level='WARNING'
        )
        # ...
```

### Intégration dans les routes

```python
# Extension des routes avec audit
@auth_bp.route('/login', methods=['POST'])
def login():
    # ...
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()

        if user and user.check_password(form.password.data) and user.is_active:
            # Audit de succès
            audit_logger.log_action(
                'LOGIN_SUCCESS',
                user=user.username,
                details={
                    'method': 'password',
                    'remember_me': form.remember_me.data
                }
            )

            login_user(user, remember=form.remember_me.data)
            # ...
        else:
            # Audit d'échec
            audit_logger.log_action(
                'LOGIN_FAILED',
                user=form.username.data,
                details={'reason': 'invalid_credentials'},
                level='WARNING'
            )
            # ...
```

## 🔒 Tests de sécurité

### Tests d'authentification

```python
# tests/test_security.py
import pytest
from werkzeug.security import check_password_hash

def test_password_hashing(app):
    """Test du hashage des mots de passe"""
    with app.app_context():
        from app.models import User

        user = User(username='test', email='test@example.com')
        user.set_password('password123')

        assert user.password_hash != 'password123'
        assert check_password_hash(user.password_hash, 'password123')
        assert not check_password_hash(user.password_hash, 'wrongpassword')

def test_account_locking(app):
    """Test du verrouillage de compte"""
    with app.app_context():
        from app.models import User

        user = User(username='test', email='test@example.com')
        user.set_password('password123')

        # Simulation d'échecs de connexion
        for i in range(5):
            user.check_password('wrongpassword')

        assert user.failed_login_attempts == 5
        assert user.is_locked()

def test_csrf_protection(client, auth_client):
    """Test de la protection CSRF"""
    # Tentative sans token CSRF
    response = client.post('/parametres', data={'theme': 'dark'})
    assert response.status_code == 400  # Bad Request

    # Avec token CSRF (via auth_client)
    response = auth_client.post('/parametres', data={'theme': 'dark'})
    assert response.status_code == 302  # Redirect success

def test_rate_limiting(client):
    """Test du rate limiting"""
    # Multiples tentatives de connexion
    for i in range(10):
        response = client.post('/auth/login', data={
            'username': 'nonexistent',
            'password': 'wrong'
        })

    # La dernière devrait être bloquée
    assert b'Too Many Requests' in response.data or response.status_code == 429
```

### Scan de sécurité

```bash
# Installation des outils
pip install safety bandit

# Vérification des vulnérabilités
safety check

# Analyse de sécurité du code
bandit -r app/

# Headers de sécurité
curl -I http://localhost:5000

# Test d'injection SQL (avec des outils comme sqlmap)
# Test XSS
# Test de la gestion des sessions
```

> **💡 À retenir** : La sécurité est un processus continu. Commencez par les bases (authentification, CSRF, HTTPS) et ajoutez des couches de protection au fur et à mesure.

> **⚠️ Astuce** : Utilisez toujours des bibliothèques éprouvées (Flask-Login, WTForms) plutôt que de réinventer la roue pour les fonctionnalités de sécurité critiques.

Félicitations ! La Partie III sur le développement de l'application Flask est maintenant complète. Vous maîtrisez maintenant la structure, l'interface utilisateur, et la sécurité de votre application. La Partie IV va explorer la connectivité et l'intégration avec des systèmes externes !

---

**Navigation**
- [Chapitre précédent : Conception de l'interface utilisateur](Chapitre_12_Interface_Utilisateur.md)
- [Partie IV : Connectivité & intégration intelligente](../Partie_IV_Connectivite_Integration/)
- [Retour à la table des matières](../../README.md)
