## ⚙️ Конфигурация проекта (Settings & Celery)

В этом разделе собраны эталонные настройки Django, DRF, Celery и переменных окружения для быстрого копирования.

### 📝 Переменные окружения (.env)
```env
# Django
SECRET_KEY=your_secret_key_here

# PostgreSQL
POSTGRES_DB=enfbd
POSTGRES_USER=enfbd
POSTGRES_PASSWORD=enfbd
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# Frontend & Services
FRONTEND_URL=http://localhost:5173
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Email
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
EMAIL_HOST=localhost
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=
EMAIL_HOST_PASSWORD=
DEFAULT_FROM_EMAIL=noreply@newssite.com

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
```

---

### 🛠 Настройки Django (settings.py)

#### База данных и окружение
```python
import os
from decouple import config # или python-dotenv

load_dotenv()

SECRET_KEY = os.getenv('SECRET_KEY')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('POSTGRES_DB'),
        'USER': os.getenv('POSTGRES_USER'),
        'PASSWORD': os.getenv('POSTGRES_PASSWORD'),
        'HOST': os.getenv('POSTGRES_HOST', 'db'),
        'PORT': os.getenv('POSTGRES_PORT', '5432'),
        'ATOMIC_REQUESTS': True,
    }
}
```

#### Статика и Медиа
```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

#### Безопасность и CORS
```python
# Настройки CORS
if DEBUG:
    CORS_ALLOW_ALL_ORIGINS = True
else:
    CORS_ALLOW_ALL_ORIGINS = [
        'http://localhost:3000',
        'http://127.0.0.1:3000',
    ]

# Безопасность
SECURE_BROWSER_XSS_FILTER = True  # Защита от XSS-атак
SECURE_CONTENT_TYPE_NOSNIFF = True # Запрет MINE-типов
X_FRAME_OPTIONS = 'Deny'            # Защита от кликджекинга
```

#### Django Rest Framework (DRF)
```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ],
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
    },
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
    ],
}
```

#### Внешние сервисы (Stripe, Email)
```python
FRONTEND_URL = config('FRONTEND_URL', default='http://localhost:5173')

# Stripe
STRIPE_PUBLISHABLE_KEY = config('STRIPE_PUBLISHABLE_KEY', default='')
STRIPE_SECRET_KEY = config('STRIPE_SECRET_KEY', default='')
STRIPE_WEBHOOK_SECRET = config('STRIPE_WEBHOOK_SECRET', default='')

# Email
EMAIL_BACKEND = config('EMAIL_BACKEND', default='django.core.mail.backends.console.EmailBackend')
EMAIL_HOST = config('EMAIL_HOST', default='localhost')
EMAIL_PORT = config('EMAIL_PORT', default=587, cast=int)
EMAIL_USE_TLS = config('EMAIL_USE_TLS', default=True, cast=bool)
EMAIL_HOST_USER = config('EMAIL_HOST_USER', default='')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD', default='')
DEFAULT_FROM_EMAIL = config('DEFAULT_FROM_EMAIL', default='noreply@newssite.com')
```

#### Celery & Celery Beat
```python
CELERY_BROKER_URL = config('CELERY_BROKER_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = config('CELERY_RESULT_BACKEND', default='redis://localhost:6379/0')
CELERY_TIMEZONE = TIME_ZONE
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']

# Периодические задачи (Beat)
CELERY_BEAT_SCHEDULE = {
    'check-expired-subscriptions': {
        'task': 'apps.subscribe.tasks.check_expired_subscriptions',
        'schedule': 3600.0, # Каждый час
    },
    'send-subscription-expiry-reminders': {
        'task': 'apps.subscribe.tasks.send_subscription_expiry_reminder',
        'schedule': 86400.0, # Каждый день
    },
    # 'cleanup-old-payments': {
    #     'task': 'apps.payment.tasks.cleanup_old_payments',
    #     'schedule': 604800.0,
    # },
    # 'cleanup-old-webhook-events': {
    #     'task': 'apps.payment.tasks.cleanup_old_webhook_events',
    #     'schedule': 86400.0,
    # },
    # 'retry-failed-webhook-events': {
    #     'task': 'apps.payment.tasks.retry_failed_webhook_events',
    #     'schedule': 3600.0,
    # },
}
```

---

### 🌐 Маршруты (urls.py)
```python
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # Ваши маршруты здесь
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

### 🦺 Инициализация Celery в проекте

#### `config/celery.py`
```python
import os
from celery import Celery

# Установка переменной окружения для Django settings
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('config')

# Настройка из settings.py с пространством имен CELERY
app.config_from_object('django.conf:settings', namespace='CELERY')

# Автоматический поиск задач в приложениях
app.autodiscover_tasks()

@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

#### `config/__init__.py`
```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```
