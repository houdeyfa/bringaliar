# Bring‑A‑Liar Night (Django 5 + DRF + Channels + Celery)

A production‑ready scaffold for a playful, ethical re‑imagining of “Le Dîner de cons.”

> **Mechanic**: A Host creates a dinner event. Each Attendee brings exactly one consenting **Liar** (performer). Liars prepare obviously false claims. During rounds, Liars perform; everyone else rates **Obviousness** and **Fun**. Points go to the Inviter. Strong consent, kindness, and moderator tools included.

---

## Quickstart

```bash
# 1) Clone & bootstrap
python -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt

# 2) Env & DB
cp .env.example .env
# edit .env as needed (SECRET_KEY, STRIPE KEYS, EMAIL, REDIS, etc.)

# 3) Migrate & seed
python manage.py migrate
python manage.py createsuperuser
python manage.py seed_demo

# 4) Run all the things (3 terminals)
# API + HTMX
python manage.py runserver
# Channels worker (Daphne optional; runserver handles ASGI, but we separate for prod)
# Example: daphne -b 0.0.0.0 -p 8001 bringaliar.asgi:application
# Celery workers
celery -A bringaliar worker -l INFO
celery -A bringaliar beat -l INFO

# 5) Visit
open http://127.0.0.1:8000/
```

**Docker (dev)**
```bash
docker compose up --build
```

---

## Project layout

```
bringaliar/
├─ manage.py
├─ requirements.txt
├─ pyproject.toml
├─ docker-compose.yml
├─ Dockerfile
├─ .env.example
├─ .env  # (local)
├─ .github/workflows/ci.yml
├─ bringaliar/
│  ├─ __init__.py
│  ├─ asgi.py
│  ├─ celery.py
│  ├─ routing.py
│  └─ settings/
│     ├─ base.py
│     ├─ dev.py
│     └─ prod.py
├─ apps/
│  ├─ core/
│  │  ├─ __init__.py
│  │  ├─ models.py
│  │  ├─ emails.py
│  │  ├─ utils.py
│  │  └─ admin.py
│  ├─ accounts/
│  │  ├─ __init__.py
│  │  ├─ models.py
│  │  ├─ adapters.py
│  │  ├─ serializers.py
│  │  ├─ admin.py
│  │  └─ urls.py
│  ├─ events/
│  │  ├─ __init__.py
│  │  ├─ models.py
│  │  ├─ serializers.py
│  │  ├─ views.py
│  │  ├─ permissions.py
│  │  ├─ signals.py
│  │  ├─ admin.py
│  │  └─ urls.py
│  ├─ pairing/
│  │  ├─ __init__.py
│  │  ├─ models.py
│  │  ├─ serializers.py
│  │  ├─ views.py
│  │  └─ admin.py
│  ├─ game/
│  │  ├─ __init__.py
│  │  ├─ models.py
│  │  ├─ services.py
│  │  ├─ tasks.py
│  │  ├─ consumers.py
│  │  ├─ serializers.py
│  │  ├─ views.py
│  │  ├─ permissions.py
│  │  └─ admin.py
│  ├─ moderation/
│  │  ├─ __init__.py
│  │  ├─ views.py
│  │  └─ permissions.py
│  └─ payments/
│     ├─ __init__.py
│     ├─ models.py
│     ├─ stripe_webhooks.py
│     ├─ views.py
│     └─ admin.py
├─ templates/
│  ├─ base.html
│  ├─ home.html
│  ├─ events/
│  │  ├─ new_wizard.html
│  │  ├─ detail.html
│  │  ├─ recap.html
│  │  └─ _partials/
│  │     ├─ roster.html
│  │     ├─ countdown.html
│  │     ├─ claim_card.html
│  │     ├─ rating_panel.html
│  │     └─ leaderboard.html
│  └─ moderation/
│     └─ queue.html
├─ static/
├─ locale/
│  ├─ en/LC_MESSAGES/django.po
│  └─ fr/LC_MESSAGES/django.po
└─ tests/
   ├─ test_scoring.py
   ├─ test_time_windows.py
   ├─ test_unique_liar.py
   └─ test_channels_api.py
```

---

## requirements.txt
```txt
Django==5.0.7
psycopg2-binary>=2.9
django-environ>=0.11

# Auth
django-allauth>=0.63

# API
djangorestframework>=3.15

# Realtime
channels>=4.1
channels-redis>=4.2

# Tasks
celery>=5.4
redis>=5.0

# Frontend
django-htmx>=1.18

# Stripe
stripe>=9.6

# Quality
ruff>=0.5
black>=24.4
mypy>=1.10
django-stubs[compatible-mypy]>=5.0
pytest>=8.2
pytest-django>=4.9

# Optional dev
python-dotenv>=1.0
```

---

## pyproject.toml (ruff, black, mypy)
```toml
[tool.black]
line-length = 100
target-version = ["py311"]

[tool.ruff]
line-length = 100
extend-select = ["I", "B", "C4", "UP", "RUF"]
ignore = ["E501"]

[tool.mypy]
python_version = "3.11"
plugins = ["mypy_django_plugin.main"]
strict = true
warn_unused_configs = true

[tool.django-stubs]
django_settings_module = "bringaliar.settings.dev"
```

---

## .env.example
```env
DJANGO_SETTINGS_MODULE=bringaliar.settings.dev
SECRET_KEY=changeme
DEBUG=1
ALLOWED_HOSTS=*
DATABASE_URL=postgres://postgres:postgres@localhost:5432/bringaliar
REDIS_URL=redis://localhost:6379/0
CHANNEL_LAYERS_URL=redis://localhost:6379/1
CELERY_BROKER_URL=redis://localhost:6379/2
CELERY_RESULT_BACKEND=redis://localhost:6379/3

# Stripe (optional)
STRIPE_PUBLIC_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Email
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
DEFAULT_FROM_EMAIL=Bring-A-Liar <noreply@bringaliar.local>

# i18n
LANGUAGE_CODE=en
TIME_ZONE=Europe/London
```

---

## bringaliar/settings/base.py
```python
from __future__ import annotations
import environ
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env(
    DEBUG=(bool, False),
    DJANGO_SETTINGS_MODULE=(str, "bringaliar.settings.dev"),
)

environ.Env.read_env(BASE_DIR.parent / ".env")

SECRET_KEY = env("SECRET_KEY")
DEBUG = env.bool("DEBUG", False)
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=["*"])

INSTALLED_APPS = [
    # Django
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django.contrib.sites",

    # Third-party
    "rest_framework",
    "channels",
    "django_htmx",
    "allauth",
    "allauth.account",

    # Local apps
    "apps.core",
    "apps.accounts",
    "apps.events",
    "apps.pairing",
    "apps.game",
    "apps.moderation",
    "apps.payments",
]

SITE_ID = 1
AUTH_USER_MODEL = "accounts.User"

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.locale.LocaleMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "django_htmx.middleware.HtmxMiddleware",
]

ROOT_URLCONF = "bringaliar.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR.parent / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ]
        },
    }
]

ASGI_APPLICATION = "bringaliar.asgi.application"
WSGI_APPLICATION = None  # ASGI only

DATABASES = {"default": env.db("DATABASE_URL", default=f"sqlite:///{BASE_DIR.parent/'db.sqlite3'}")}

# Channels
REDIS_URL = env("CHANNEL_LAYERS_URL", default=env("REDIS_URL", default="redis://localhost:6379/1"))
CHANNEL_LAYERS = {
    "default": {"BACKEND": "channels_redis.core.RedisChannelLayer", "CONFIG": {"hosts": [REDIS_URL]}},
}

# Celery
CELERY_BROKER_URL = env("CELERY_BROKER_URL", default=env("REDIS_URL"))
CELERY_RESULT_BACKEND = env("CELERY_RESULT_BACKEND", default=env("REDIS_URL"))
CELERY_TIMEZONE = env("TIME_ZONE", default="Europe/London")

# Static & media
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR.parent / "staticfiles"
STATICFILES_DIRS = [BASE_DIR.parent / "static"]

# i18n
LANGUAGE_CODE = env("LANGUAGE_CODE", default="en")
TIME_ZONE = env("TIME_ZONE", default="Europe/London")
USE_I18N = True
USE_TZ = True

# DRF
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
}

# Allauth (email + password; magic link optional via adapters)
AUTHENTICATION_BACKENDS = (
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
)
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_AUTHENTICATION_METHOD = "email"
ACCOUNT_EMAIL_VERIFICATION = "mandatory"
LOGIN_REDIRECT_URL = "/"
LOGOUT_REDIRECT_URL = "/"

DEFAULT_FROM_EMAIL = env("DEFAULT_FROM_EMAIL", default="Bring-A-Liar <noreply@bringaliar.local>")
EMAIL_BACKEND = env("EMAIL_BACKEND", default="django.core.mail.backends.console.EmailBackend")
```

### bringaliar/settings/dev.py
```python
from .base import *  # noqa
DEBUG = True
CSRF_TRUSTED_ORIGINS = ["http://127.0.0.1:8000", "http://localhost:8000"]
```

### bringaliar/settings/prod.py
```python
from .base import *  # noqa
DEBUG = False
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SECURE_HSTS_SECONDS = 3600
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
```

---

## bringaliar/asgi.py & routing.py
```python
# asgi.py
from __future__ import annotations
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from django.urls import path
from apps.game.consumers import EventConsumer

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "bringaliar.settings.dev")

django_asgi_app = get_asgi_application()

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddlewareStack(
        URLRouter([
            path("ws/events/<int:event_id>/", EventConsumer.as_asgi(), name="ws_event"),
        ])
    ),
})
```

```python
# routing.py (kept for future split routes)
from __future__ import annotations
from django.urls import path
from apps.game.consumers import EventConsumer

websocket_urlpatterns = [
    path("ws/events/<int:event_id>/", EventConsumer.as_asgi()),
]
```

---

## bringaliar/celery.py
```python
from __future__ import annotations
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "bringaliar.settings.dev")

app = Celery("bringaliar")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# bringaliar/__init__.py
default_app_config = "apps.core.apps.CoreConfig"
from .celery import app as celery_app  # noqa: E402
__all__ = ("celery_app",)
```

---

## apps/core/models.py (Badge, UserBadge, Consent, mixins)
```python
from __future__ import annotations
from django.conf import settings
from django.db import models
from django.utils import timezone

class TimeStamped(models.Model):
    created_at = models.DateTimeField(default=timezone.now, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class Badge(TimeStamped):
    code = models.SlugField(unique=True)
    name = models.CharField(max_length=120)
    description = models.TextField(blank=True)

    def __str__(self) -> str:
        return self.name

class UserBadge(TimeStamped):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    badge = models.ForeignKey(Badge, on_delete=models.CASCADE)
    awarded_at = models.DateTimeField(default=timezone.now)

    class Meta:
        unique_together = ("user", "badge")

class Consent(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    code_of_conduct_accepted_at = models.DateTimeField(null=True, blank=True)
    data_consent_at = models.DateTimeField(null=True, blank=True)

    def mark(self, coc: bool = False, data: bool = False) -> None:
        now = timezone.now()
        if coc:
            self.code_of_conduct_accepted_at = now
        if data:
            self.data_consent_at = now
        self.save(update_fields=["code_of_conduct_accepted_at", "data_consent_at"])
```

---

## apps/accounts/models.py (Custom User)
```python
from __future__ import annotations
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    class Role(models.TextChoices):
        HOST = "host", "Host"
        ATTENDEE = "attendee", "Attendee"
        LIAR = "liar", "Liar"
        MODERATOR = "moderator", "Moderator"

    # Note: actual permissions via Groups/Permissions; this is a preferred role
    role = models.CharField(max_length=16, choices=Role.choices, default=Role.ATTENDEE)
    locale = models.CharField(max_length=12, default="en")

    def display_name(self) -> str:
        return self.get_full_name() or self.email or self.username
```

---

## apps/events/models.py (Event, Attendee)
```python
from __future__ import annotations
from django.conf import settings
from django.db import models
from django.utils import timezone

class Event(models.Model):
    class Visibility(models.TextChoices):
        PUBLIC = "public", "Public"
        PRIVATE = "private", "Private"

    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"
        CLOSED = "closed", "Closed"

    host = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="hosted_events")
    title = models.CharField(max_length=140)
    description = models.TextField(blank=True)
    start_at = models.DateTimeField()
    end_at = models.DateTimeField()
    location = models.CharField(max_length=240, blank=True)
    is_online = models.BooleanField(default=False)
    max_attendees = models.PositiveIntegerField(default=12)
    price_cents = models.PositiveIntegerField(default=0)
    currency = models.CharField(max_length=8, default="GBP")
    visibility = models.CharField(max_length=10, choices=Visibility.choices, default=Visibility.PRIVATE)
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)

    created_at = models.DateTimeField(default=timezone.now, db_index=True)

    class Meta:
        indexes = [
            models.Index(fields=["start_at"]),
            models.Index(fields=["visibility", "status"]),
        ]

    def __str__(self) -> str:
        return f"{self.title} ({self.start_at:%Y-%m-%d})"

class Attendee(models.Model):
    class RSVP(models.TextChoices):
        INVITED = "invited", "Invited"
        GOING = "going", "Going"
        MAYBE = "maybe", "Maybe"
        DECLINED = "declined", "Declined"

    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name="attendees")
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="attendances")
    rsvp_status = models.CharField(max_length=10, choices=RSVP.choices, default=RSVP.INVITED)
    paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(default=timezone.now)

    class Meta:
        unique_together = ("event", "user")

    def __str__(self) -> str:
        return f"{self.user_id}@{self.event_id}:{self.rsvp_status}"
```

---

## apps/pairing/models.py (LiarProfile)
```python
from __future__ import annotations
from django.db import models
from django.utils import timezone
from apps.events.models import Event, Attendee

class LiarProfile(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name="liars")
    inviter = models.OneToOneField(Attendee, on_delete=models.CASCADE, related_name="liar_profile")
    display_name = models.CharField(max_length=80)
    persona_bio = models.TextField(blank=True)
    consent_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = ("event", "inviter")

    def give_consent(self) -> None:
        self.consent_at = timezone.now()
        self.save(update_fields=["consent_at"])
```

**Constraint**: One Liar per Attendee enforced via `OneToOneField(inviter)`.

---

## apps/game/models.py (Round, Claim, Rating, Score)
```python
from __future__ import annotations
from statistics import median
from django.db import models
from django.utils import timezone
from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile

class Round(models.Model):
    class Phase(models.TextChoices):
        PREP = "prep", "Preparation"
        PERFORM = "perform", "Perform"
        RATE = "rate", "Rate"
        CLOSED = "closed", "Closed"

    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name="rounds")
    index = models.PositiveIntegerField()
    phase = models.CharField(max_length=8, choices=Phase.choices, default=Phase.PREP)
    perform_start_at = models.DateTimeField(null=True, blank=True)
    rate_close_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = ("event", "index")
        indexes = [models.Index(fields=["event", "index"])]

class Claim(models.Model):
    CATEGORY_CHOICES = [
        ("work", "Work"),
        ("hobby", "Hobby"),
        ("skill", "Skill"),
        ("achievement", "Achievement"),
        ("other", "Other"),
    ]
    round = models.ForeignKey(Round, on_delete=models.CASCADE, related_name="claims")
    liar = models.ForeignKey(LiarProfile, on_delete=models.CASCADE, related_name="claims")
    content = models.CharField(max_length=280)
    category = models.CharField(max_length=20, choices=CATEGORY_CHOICES, default="other")
    performed_at = models.DateTimeField(null=True, blank=True)
    is_flagged = models.BooleanField(default=False)
    flag_reason = models.TextField(blank=True, null=True)

    # Computed aggregates (denormalized for recap)
    obviousness_index = models.PositiveIntegerField(default=0)
    fun_index = models.PositiveIntegerField(default=0)
    facepalm_score = models.PositiveIntegerField(default=0)

class Rating(models.Model):
    claim = models.ForeignKey(Claim, on_delete=models.CASCADE, related_name="ratings")
    rater = models.ForeignKey(Attendee, on_delete=models.CASCADE)
    obviousness = models.PositiveSmallIntegerField()
    fun = models.PositiveSmallIntegerField()
    created_at = models.DateTimeField(default=timezone.now)

    class Meta:
        unique_together = ("claim", "rater")

class Score(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name="scores")
    attendee = models.ForeignKey(Attendee, on_delete=models.CASCADE, related_name="scores")
    points = models.IntegerField(default=0)

    class Meta:
        unique_together = ("event", "attendee")
```

---

## apps/game/services.py (scoring)
```python
from __future__ import annotations
from math import ceil
from statistics import median
from django.db.models import QuerySet
from apps.game.models import Claim, Rating, Score, Round
from apps.pairing.models import LiarProfile

OBV_MAX = 5
FUN_MAX = 5

def _pct(val: float, denom: int) -> int:
    return round(100 * val / denom) if denom else 0

def compute_claim_scores(claim: Claim) -> Claim:
    ratings: QuerySet[Rating] = claim.ratings.all()
    if not ratings.exists():
        claim.obviousness_index = 0
        claim.fun_index = 0
        claim.facepalm_score = 0
        claim.save(update_fields=["obviousness_index", "fun_index", "facepalm_score"])
        return claim

    obv_med = median(r.obviousness for r in ratings)
    fun_med = median(r.fun for r in ratings)

    obviousness_index = _pct(obv_med, OBV_MAX)
    fun_index = _pct(fun_med, FUN_MAX)
    facepalm_score = round(0.7 * obviousness_index + 0.3 * fun_index)

    claim.obviousness_index = int(obviousness_index)
    claim.fun_index = int(fun_index)
    claim.facepalm_score = int(facepalm_score)
    claim.save(update_fields=["obviousness_index", "fun_index", "facepalm_score"])
    return claim

def inviter_points_for_claim(claim: Claim) -> int:
    ratings = list(claim.ratings.all())
    if not ratings:
        return 0
    points = ceil(claim.facepalm_score / 10)
    # Whopper bonus: >=70% of raters chose 5 for obviousness
    if ratings and (sum(1 for r in ratings if r.obviousness == 5) / len(ratings)) >= 0.7:
        points += 2
    return int(points)

def recompute_scores_for_round(rnd: Round) -> None:
    # Aggregate per inviter/attendee
    event = rnd.event
    # ensure Score rows exist for all attendees
    from apps.events.models import Attendee
    for att in Attendee.objects.filter(event=event):
        Score.objects.get_or_create(event=event, attendee=att)

    # recompute per claim then add to inviter
    for claim in rnd.claims.select_related("liar", "liar__inviter").all():
        compute_claim_scores(claim)
        inviter = claim.liar.inviter
        s, _ = Score.objects.get_or_create(event=event, attendee=inviter)
        s.points += inviter_points_for_claim(claim)
        s.save(update_fields=["points"])
```

---

## apps/game/tasks.py (Celery round progression)
```python
from __future__ import annotations
from datetime import timedelta
from django.utils import timezone
from celery import shared_task
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from apps.game.models import Round
from apps.game.services import recompute_scores_for_round

@shared_task
def start_perform_phase(round_id: int) -> None:
    rnd = Round.objects.select_related("event").get(id=round_id)
    rnd.phase = Round.Phase.PERFORM
    rnd.perform_start_at = timezone.now()
    rnd.save(update_fields=["phase", "perform_start_at"])
    _broadcast(rnd, {"type": "phase", "phase": rnd.phase})

@shared_task
def open_rate_phase(round_id: int, duration_s: int = 45) -> None:
    rnd = Round.objects.select_related("event").get(id=round_id)
    rnd.phase = Round.Phase.RATE
    rnd.rate_close_at = timezone.now() + timedelta(seconds=duration_s)
    rnd.save(update_fields=["phase", "rate_close_at"])
    _broadcast(rnd, {"type": "phase", "phase": rnd.phase, "rate_close_at": rnd.rate_close_at.isoformat()})

@shared_task
def close_round(round_id: int) -> None:
    rnd = Round.objects.select_related("event").get(id=round_id)
    rnd.phase = Round.Phase.CLOSED
    rnd.save(update_fields=["phase"])
    recompute_scores_for_round(rnd)
    _broadcast(rnd, {"type": "phase", "phase": rnd.phase, "leaderboard": True})

@shared_task
def countdown_tick(round_id: int) -> None:
    # Optional ticker if you want per-second updates
    rnd = Round.objects.get(id=round_id)
    if rnd.rate_close_at:
        remaining = int((rnd.rate_close_at - timezone.now()).total_seconds())
        _broadcast(rnd, {"type": "tick", "remaining": max(0, remaining)})

# Helpers

def _broadcast(rnd: Round, payload: dict) -> None:
    layer = get_channel_layer()
    async_to_sync(layer.group_send)(f"event_{rnd.event_id}", {"type": "broadcast", "payload": payload})
```

---

## apps/game/consumers.py (Channels consumer)
```python
from __future__ import annotations
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class EventConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.event_id = self.scope["url_route"]["kwargs"]["event_id"]
        self.group_name = f"event_{self.event_id}"
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def receive_json(self, content, **kwargs):
        # Clients are passive; server pushes updates.
        pass

    async def broadcast(self, event):
        await self.send_json(event["payload"])
```

---

## apps/events/serializers.py & views.py (DRF API)
```python
# serializers.py
from __future__ import annotations
from rest_framework import serializers
from .models import Event, Attendee

class EventSerializer(serializers.ModelSerializer):
    host_email = serializers.EmailField(source="host.email", read_only=True)

    class Meta:
        model = Event
        fields = (
            "id", "title", "description", "start_at", "end_at", "location", "is_online",
            "max_attendees", "price_cents", "currency", "visibility", "status", "host_email",
        )

class AttendeeSerializer(serializers.ModelSerializer):
    user_email = serializers.EmailField(source="user.email", read_only=True)

    class Meta:
        model = Attendee
        fields = ("id", "event", "user", "rsvp_status", "paid", "user_email")
        read_only_fields = ("paid",)
```

```python
# views.py
from __future__ import annotations
from django.shortcuts import get_object_or_404
from rest_framework import permissions, status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Event, Attendee
from .serializers import EventSerializer, AttendeeSerializer

class IsHostOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj: Event):
        if request.method in permissions.SAFE_METHODS:
            return True
        return getattr(obj, "host_id", None) == getattr(request.user, "id", None)

class EventViewSet(viewsets.ModelViewSet):
    queryset = Event.objects.all()
    serializer_class = EventSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsHostOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(host=self.request.user)

    @action(detail=True, methods=["patch"], url_path="publish")
    def publish(self, request, pk=None):
        event = self.get_object()
        event.status = Event.Status.PUBLISHED
        event.save(update_fields=["status"])
        return Response({"status": event.status})

    @action(detail=True, methods=["patch"], url_path="close")
    def close(self, request, pk=None):
        event = self.get_object()
        event.status = Event.Status.CLOSED
        event.save(update_fields=["status"])
        return Response({"status": event.status})

    @action(detail=True, methods=["get"], url_path="leaderboard")
    def leaderboard(self, request, pk=None):
        from apps.game.models import Score
        scores = Score.objects.filter(event_id=pk).select_related("attendee__user").order_by("-points")
        data = [{"attendee_id": s.attendee_id, "email": s.attendee.user.email, "points": s.points} for s in scores]
        return Response(data)

class RSVPViewSet(viewsets.GenericViewSet):
    queryset = Attendee.objects.all()
    serializer_class = AttendeeSerializer

    @action(detail=False, methods=["post"], url_path=r"events/(?P<event_id>[^/.]+)/rsvp")
    def rsvp(self, request, event_id=None):
        event = get_object_or_404(Event, id=event_id)
        att, _ = Attendee.objects.get_or_create(event=event, user=request.user)
        status_value = request.data.get("rsvp_status", Attendee.RSVP.GOING)
        att.rsvp_status = status_value
        att.save(update_fields=["rsvp_status"])
        return Response(AttendeeSerializer(att).data)
```

**urls.py (project)**
```python
from __future__ import annotations
from django.contrib import admin
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from apps.events.views import EventViewSet, RSVPViewSet
from apps.game.views import RoundViewSet, ClaimViewSet, RatingViewSet
from apps.moderation.views import ModerationView

router = DefaultRouter()
router.register(r"events", EventViewSet, basename="event")
router.register(r"rsvp", RSVPViewSet, basename="rsvp")
router.register(r"rounds", RoundViewSet, basename="round")
router.register(r"claims", ClaimViewSet, basename="claim")
router.register(r"ratings", RatingViewSet, basename="rating")

urlpatterns = [
    path("admin/", admin.site.urls),
    path("accounts/", include("allauth.urls")),
    path("api/", include(router.urls)),
    path("moderation/", ModerationView.as_view(), name="moderation"),
    path("", include("apps.events.urls")),  # HTMX pages
]
```

---

## apps/game/serializers.py & views.py (rounds, claims, rating)
```python
# serializers.py
from __future__ import annotations
from rest_framework import serializers
from .models import Round, Claim, Rating

class RoundSerializer(serializers.ModelSerializer):
    class Meta:
        model = Round
        fields = ("id", "event", "index", "phase", "perform_start_at", "rate_close_at")

class ClaimSerializer(serializers.ModelSerializer):
    class Meta:
        model = Claim
        fields = (
            "id", "round", "liar", "content", "category", "performed_at",
            "is_flagged", "flag_reason", "obviousness_index", "fun_index", "facepalm_score",
        )

class RatingSerializer(serializers.ModelSerializer):
    class Meta:
        model = Rating
        fields = ("id", "claim", "rater", "obviousness", "fun", "created_at")
        read_only_fields = ("created_at",)
        extra_kwargs = {
            "obviousness": {"min_value": 1, "max_value": 5},
            "fun": {"min_value": 1, "max_value": 5},
        }
```

```python
# views.py
from __future__ import annotations
from datetime import timedelta
from django.utils import timezone
from django.shortcuts import get_object_or_404
from rest_framework import permissions, status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Round, Claim, Rating
from .serializers import RoundSerializer, ClaimSerializer, RatingSerializer
from .tasks import start_perform_phase, open_rate_phase, close_round
from .services import compute_claim_scores
from apps.events.models import Attendee, Event

class IsEventMember(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        event = obj if isinstance(obj, Event) else getattr(obj, "event", None)
        if not event:
            return False
        return Attendee.objects.filter(event=event, user=request.user, rsvp_status=Attendee.RSVP.GOING).exists() or event.host_id == request.user.id

class RoundViewSet(viewsets.ModelViewSet):
    queryset = Round.objects.all()
    serializer_class = RoundSerializer
    permission_classes = [permissions.IsAuthenticated]

    @action(detail=True, methods=["post"], url_path="start")
    def start(self, request, pk=None):
        rnd = self.get_object()
        start_perform_phase.delay(rnd.id)
        return Response({"ok": True})

    @action(detail=True, methods=["post"], url_path="advance")
    def advance(self, request, pk=None):
        rnd = self.get_object()
        # perform -> rate -> closed
        if rnd.phase == Round.Phase.PERFORM:
            open_rate_phase.delay(rnd.id)
        elif rnd.phase == Round.Phase.RATE:
            close_round.delay(rnd.id)
        return Response({"phase": rnd.phase})

class ClaimViewSet(viewsets.ModelViewSet):
    queryset = Claim.objects.all()
    serializer_class = ClaimSerializer
    permission_classes = [permissions.IsAuthenticated]

class RatingViewSet(viewsets.GenericViewSet):
    queryset = Rating.objects.all()
    serializer_class = RatingSerializer

    @action(detail=False, methods=["post"], url_path=r"claims/(?P<claim_id>[^/.]+)/rate")
    def rate(self, request, claim_id=None):
        claim = get_object_or_404(Claim, id=claim_id)
        att = get_object_or_404(Attendee, event=claim.round.event, user=request.user)
        # Exclude the inviter from rating their own liar's claim
        if claim.liar.inviter_id == att.id:
            return Response({"detail": "You cannot rate your invited Liar."}, status=400)

        # Time window: only during RATE phase and before close
        rnd = claim.round
        now = timezone.now()
        if not (rnd.phase == Round.Phase.RATE and (rnd.rate_close_at is None or now <= rnd.rate_close_at)):
            return Response({"detail": "Ratings are closed."}, status=400)

        data = {
            "claim": claim.id,
            "rater": att.id,
            "obviousness": int(request.data.get("obviousness", 3)),
            "fun": int(request.data.get("fun", 3)),
        }
        rating, created = Rating.objects.update_or_create(
            claim=claim, rater=att, defaults={"obviousness": data["obviousness"], "fun": data["fun"]}
        )
        compute_claim_scores(claim)
        return Response(RatingSerializer(rating).data, status=201 if created else 200)
```

---

## apps/moderation/views.py (flag queue, pause/extend stubs)
```python
from __future__ import annotations
from django.views.generic import TemplateView
from django.contrib.auth.mixins import LoginRequiredMixin
from apps.game.models import Claim

class ModerationView(LoginRequiredMixin, TemplateView):
    template_name = "moderation/queue.html"

    def get_context_data(self, **kwargs):
        ctx = super().get_context_data(**kwargs)
        ctx["flagged"] = Claim.objects.filter(is_flagged=True).select_related("liar__inviter__user")
        return ctx
```

---

## apps/payments/models.py & stripe_webhooks.py (optional payments)
```python
from __future__ import annotations
from django.conf import settings
from django.db import models
from django.utils import timezone
from apps.events.models import Event

class Payment(models.Model):
    PROVIDERS = [("stripe", "Stripe")]

    event = models.ForeignKey(Event, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    provider = models.CharField(max_length=20, choices=PROVIDERS, default="stripe")
    status = models.CharField(max_length=20, default="created")
    amount_cents = models.PositiveIntegerField(default=0)
    currency = models.CharField(max_length=8, default="GBP")
    created_at = models.DateTimeField(default=timezone.now)
    raw_payload = models.JSONField(default=dict, blank=True)
```

```python
# stripe_webhooks.py (minimal)
from __future__ import annotations
import stripe
from django.conf import settings
from django.http import HttpRequest, HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from apps.events.models import Attendee
from .models import Payment

@csrf_exempt
def stripe_webhook(request: HttpRequest) -> HttpResponse:
    payload = request.body
    sig_header = request.headers.get("Stripe-Signature")
    endpoint_secret = settings.STRIPE_WEBHOOK_SECRET
    try:
        event = stripe.Webhook.construct_event(payload, sig_header, endpoint_secret)
    except Exception:
        return HttpResponse(status=400)

    if event["type"] == "checkout.session.completed":
        data = event["data"]["object"]
        # You'd map session -> event/user here (metadata)
        # att = Attendee.objects.get(...)
        # att.paid = True; att.save(update_fields=["paid"])
        Payment.objects.create(status="succeeded", raw_payload=data)

    return JsonResponse({"received": True})
```

---

## Minimal HTMX templates (Tailwind‑ready)

### templates/base.html
```html
<!doctype html>
<html lang="en" class="h-full">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Bring‑A‑Liar Night</title>
  <script src="https://unpkg.com/htmx.org@1.9.12"></script>
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@3.4.7/dist/tailwind.min.css" rel="stylesheet" />
</head>
<body class="min-h-full bg-gray-50 text-gray-900">
  <header class="px-6 py-4 border-b bg-white">
    <div class="max-w-5xl mx-auto flex items-center justify-between">
      <a href="/" class="font-extrabold text-xl">Bring‑A‑Liar</a>
      <nav class="space-x-4">
        <a href="/" class="hover:underline">Home</a>
        <a href="/events/new" class="hover:underline">Create Event</a>
        <a href="/accounts/login/" class="hover:underline">Login</a>
      </nav>
    </div>
  </header>
  <main class="max-w-5xl mx-auto p-6">
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

### templates/home.html
```html
{% extends 'base.html' %}
{% block content %}
  <div class="prose max-w-none">
    <h1>Bring‑A‑Liar Night</h1>
    <p class="text-sm">Humor, consent, and kindness first. ✨</p>
    <a class="inline-block px-4 py-2 rounded bg-black text-white" href="/events/new">Host a Night</a>
  </div>
  <h2 class="mt-8 font-semibold">Public Events</h2>
  <div id="public-events" hx-get="/api/events/?visibility=public&status=published" hx-trigger="load">
    <!-- Render via DRF (or hydrate with a tiny view) -->
  </div>
{% endblock %}
```

### templates/events/detail.html (lobby excerpt)
```html
{% extends 'base.html' %}
{% block content %}
  <h1 class="text-2xl font-bold">{{ event.title }}</h1>
  <p class="text-gray-600">{{ event.start_at }} — {{ event.location }}{% if event.is_online %} (Online){% endif %}</p>

  <div id="roster" hx-get="{% url 'event_roster' event.id %}" hx-trigger="load, every 5s">
    Loading roster…
  </div>

  {% if is_host %}
  <form method="post" action="/api/rounds/" class="mt-4">
    {% csrf_token %}
    <input type="hidden" name="event" value="{{ event.id }}" />
    <input type="hidden" name="index" value="{{ next_index }}" />
    <button class="px-4 py-2 rounded bg-indigo-600 text-white">Start Round {{ next_index }}</button>
  </form>
  {% endif %}

  <div class="mt-6">
    <h2 class="font-semibold">Live</h2>
    <div id="countdown" hx-get="{% url 'event_countdown' event.id %}" hx-trigger="load, every 1s"></div>
  </div>
{% endblock %}
```

### templates/events/_partials/countdown.html
```html
{% if remaining %}
  <div class="text-lg">⏳ {{ remaining }}s</div>
{% else %}
  <div>Waiting…</div>
{% endif %}
```

---

## apps/events/urls.py (HTMX helpers)
```python
from __future__ import annotations
from django.urls import path
from django.shortcuts import render, get_object_or_404
from django.http import HttpRequest
from .models import Event, Attendee
from apps.game.models import Round

urlpatterns = []

def home(request: HttpRequest):
    return render(request, "home.html")

urlpatterns += [path("", home, name="home")]

# Lobby page
urlpatterns += [
    path("events/<int:event_id>/", lambda r, event_id: event_detail(r, event_id), name="event_detail"),
]

def event_detail(request: HttpRequest, event_id: int):
    ev = get_object_or_404(Event, id=event_id)
    next_index = (ev.rounds.count() or 0) + 1
    return render(request, "events/detail.html", {"event": ev, "next_index": next_index, "is_host": ev.host_id == request.user.id})

# HTMX roster partial
urlpatterns += [
    path("events/<int:event_id>/roster/", lambda r, event_id: roster(r, event_id), name="event_roster"),
]

def roster(request: HttpRequest, event_id: int):
    ev = get_object_or_404(Event, id=event_id)
    attendees = ev.attendees.select_related("user").all()
    return render(request, "events/_partials/roster.html", {"attendees": attendees})

# HTMX countdown
urlpatterns += [
    path("events/<int:event_id>/countdown/", lambda r, event_id: countdown(r, event_id), name="event_countdown"),
]

def countdown(request: HttpRequest, event_id: int):
    rnd = Round.objects.filter(event_id=event_id).order_by("-index").first()
    remaining = None
    if rnd and rnd.rate_close_at:
        from django.utils import timezone
        remaining = max(0, int((rnd.rate_close_at - timezone.now()).total_seconds()))
    return render(request, "events/_partials/countdown.html", {"remaining": remaining})
```

---

## Management command: seed_demo
```python
# apps/core/management/commands/seed_demo.py
from __future__ import annotations
from django.core.management.base import BaseCommand
from django.utils import timezone
from django.contrib.auth import get_user_model
from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile

class Command(BaseCommand):
    help = "Seed demo data"

    def handle(self, *args, **opts):
        User = get_user_model()
        host, _ = User.objects.get_or_create(email="host@example.com", defaults={"username": "host", "role": "host"})
        event, _ = Event.objects.get_or_create(
            host=host,
            title="Demo Bring‑A‑Liar Night",
            defaults={
                "description": "A playful demo",
                "start_at": timezone.now() + timezone.timedelta(days=1),
                "end_at": timezone.now() + timezone.timedelta(days=1, hours=2),
                "location": "London",
                "visibility": Event.Visibility.PUBLIC,
                "status": Event.Status.PUBLISHED,
            },
        )
        for i in range(1, 5):
            u, _ = User.objects.get_or_create(email=f"guest{i}@example.com", defaults={"username": f"guest{i}"})
            att, _ = Attendee.objects.get_or_create(event=event, user=u, defaults={"rsvp_status": Attendee.RSVP.GOING})
            LiarProfile.objects.get_or_create(event=event, inviter=att, defaults={"display_name": f"Liar {i}", "persona_bio": "Ridiculous astronaut-chef."})
        self.stdout.write(self.style.SUCCESS("Seeded demo data."))
```

---

## tests (high‑value examples)

### tests/test_scoring.py
```python
import pytest
from django.utils import timezone
from django.contrib.auth import get_user_model
from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile
from apps.game.models import Round, Claim, Rating, Score
from apps.game.services import compute_claim_scores, inviter_points_for_claim, recompute_scores_for_round

def setup_event(db):
    U = get_user_model()
    host = U.objects.create(username="h", email="h@example.com")
    e = Event.objects.create(host=host, title="T", start_at=timezone.now(), end_at=timezone.now())
    a1 = Attendee.objects.create(event=e, user=U.objects.create(username="a1", email="a1@x.com"), rsvp_status=Attendee.RSVP.GOING)
    a2 = Attendee.objects.create(event=e, user=U.objects.create(username="a2", email="a2@x.com"), rsvp_status=Attendee.RSVP.GOING)
    l1 = LiarProfile.objects.create(event=e, inviter=a1, display_name="L", persona_bio="B")
    rnd = Round.objects.create(event=e, index=1)
    return e, a1, a2, l1, rnd

@pytest.mark.django_db
def test_scoring_whopper_bonus():
    e, a1, a2, l1, rnd = setup_event(None)
    claim = Claim.objects.create(round=rnd, liar=l1, content="I invented rain.")
    Rating.objects.create(claim=claim, rater=a2, obviousness=5, fun=4)
    compute_claim_scores(claim)
    pts = inviter_points_for_claim(claim)
    assert pts >= 2  # whopper bonus kicks in with 70% 5s; with one rater 100% -> +2

@pytest.mark.django_db
def test_recompute_scores_updates_event_scores():
    e, a1, a2, l1, rnd = setup_event(None)
    claim = Claim.objects.create(round=rnd, liar=l1, content="I ate the moon.")
    Rating.objects.create(claim=claim, rater=a2, obviousness=5, fun=5)
    recompute_scores_for_round(rnd)
    s = Score.objects.get(event=e, attendee=a1)
    assert s.points > 0
```

### tests/test_unique_liar.py
```python
import pytest
from django.utils import timezone
from django.contrib.auth import get_user_model
from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile

@pytest.mark.django_db
def test_one_liar_per_attendee():
    U = get_user_model()
    host = U.objects.create(username="h", email="h@x.com")
    e = Event.objects.create(host=host, title="T", start_at=timezone.now(), end_at=timezone.now())
    a = Attendee.objects.create(event=e, user=U.objects.create(username="u", email="u@x.com"))
    LiarProfile.objects.create(event=e, inviter=a, display_name="L1")
    with pytest.raises(Exception):
        LiarProfile.objects.create(event=e, inviter=a, display_name="L2")
```

### tests/test_time_windows.py
```python
import pytest
from django.utils import timezone
from django.contrib.auth import get_user_model
from rest_framework.test import APIClient
from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile
from apps.game.models import Round, Claim

@pytest.mark.django_db
def test_rating_closed_outside_rate_phase():
    U = get_user_model()
    u1 = U.objects.create(username="u1", email="u1@x.com")
    u2 = U.objects.create(username="u2", email="u2@x.com")
    e = Event.objects.create(host=u1, title="T", start_at=timezone.now(), end_at=timezone.now())
    a1 = Attendee.objects.create(event=e, user=u1, rsvp_status=Attendee.RSVP.GOING)
    a2 = Attendee.objects.create(event=e, user=u2, rsvp_status=Attendee.RSVP.GOING)
    l1 = LiarProfile.objects.create(event=e, inviter=a1, display_name="L1")
    rnd = Round.objects.create(event=e, index=1, phase=Round.Phase.PERFORM)
    c = Claim.objects.create(round=rnd, liar=l1, content="Claim")

    client = APIClient()
    client.force_authenticate(user=u2)
    resp = client.post(f"/api/ratings/claims/{c.id}/rate/", {"obviousness": 5, "fun": 5})
    assert resp.status_code == 400
```

---

## Docker & CI

### docker-compose.yml
```yaml
version: "3.9"
services:
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes: [".:/app"]
    env_file: .env
    ports: ["8000:8000"]
    depends_on: [db, redis]
  worker:
    build: .
    command: celery -A bringaliar worker -l INFO
    volumes: [".:/app"]
    env_file: .env
    depends_on: [db, redis]
  beat:
    build: .
    command: celery -A bringaliar beat -l INFO
    volumes: [".:/app"]
    env_file: .env
    depends_on: [db, redis]
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: bringaliar
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

### Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN apt-get update && apt-get install -y build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

### .github/workflows/ci.yml
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd="pg_isready -U postgres" --health-interval=10s --health-timeout=5s --health-retries=5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
      REDIS_URL: redis://localhost:6379/0
      CHANNEL_LAYERS_URL: redis://localhost:6379/1
      SECRET_KEY: test
      DJANGO_SETTINGS_MODULE: bringaliar.settings.dev
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt
      - run: python manage.py migrate --noinput
      - run: pytest -q
      - run: ruff check .
      - run: black --check .
      - run: mypy .
```

---

## Safety, consent, kindness — notes
- **Mandatory consent**: `LiarProfile.consent_at` must be set (enforce in onboarding view before allowing performance).
- **Moderator tools**: Flag/hide claims, pause rounds by not advancing, extend timers by resetting `rate_close_at`.
- **Privacy**: Only event members can view content. Provide host control to delete claims/recap on request.
- **Filters**: Add simple profanity filter in form clean (omitted here for brevity — integrate a curated list or `better-profanity`).

---

## Next steps & polish
- Add Tailwind build pipeline (or `django-tailwind`) for custom theme.
- Implement awards calculation at recap (Most Outrageous, Deadpan Disaster, Crowd Pleaser) using claim metrics.
- Add "magic link" via allauth adapter or a tiny signed‑token view.
- Enrich Channels: push roster updates on RSVP; push leaderboard diffs.
- Stripe Checkout session creation + metadata mapping to (event_id, user_id).
- i18n strings (`gettext_lazy`) and `locale` files.
- Playwright E2E covering the full happy path.

---

### License
MIT (adjust as needed).


---

### tests/test_unique_liar_v2.py
```python
"""
Extended tests for LiarProfile uniqueness.

This file complements the simpler `tests/test_unique_liar.py` you already have
(in the same repo). It keeps behavior consistent with the current models:
- `LiarProfile.inviter` is a OneToOneField to Attendee (one liar per attendee).
- Each Attendee is unique per (event, user), so the same user can have a different
  Attendee (and thus a LiarProfile) in another event.
"""

import pytest
from django.db import IntegrityError
from django.utils import timezone
from django.contrib.auth import get_user_model

from apps.events.models import Event, Attendee
from apps.pairing.models import LiarProfile


@pytest.mark.django_db
def test_second_liar_for_same_attendee_raises_integrity_error():
    """Matches the intention of tests/test_unique_liar.py but asserts IntegrityError explicitly."""
    U = get_user_model()
    host = U.objects.create(username="host", email="host@example.com")
    event = Event.objects.create(
        host=host,
        title="Night",
        start_at=timezone.now(),
        end_at=timezone.now(),
    )
    user = U.objects.create(username="alice", email="alice@example.com")
    att = Attendee.objects.create(event=event, user=user, rsvp_status=Attendee.RSVP.GOING)

    LiarProfile.objects.create(event=event, inviter=att, display_name="Alice's Liar")
    with pytest.raises(IntegrityError):
        # OneToOne(inviter) forbids another profile for the same Attendee
        LiarProfile.objects.create(event=event, inviter=att, display_name="Duplicate")


@pytest.mark.django_db
def test_two_attendees_same_event_each_can_have_a_liar():
    U = get_user_model()
    host = U.objects.create(username="h", email="h@example.com")
    e = Event.objects.create(host=host, title="T", start_at=timezone.now(), end_at=timezone.now())

    u1 = U.objects.create(username="u1", email="u1@example.com")
    u2 = U.objects.create(username="u2", email="u2@example.com")
    a1 = Attendee.objects.create(event=e, user=u1, rsvp_status=Attendee.RSVP.GOING)
    a2 = Attendee.objects.create(event=e, user=u2, rsvp_status=Attendee.RSVP.GOING)

    l1 = LiarProfile.objects.create(event=e, inviter=a1, display_name="L1")
    l2 = LiarProfile.objects.create(event=e, inviter=a2, display_name="L2")

    assert l1.inviter_id == a1.id
    assert l2.inviter_id == a2.id


@pytest.mark.django_db
def test_same_user_different_events_can_have_separate_liars():
    """Because Attendee is per (event, user), the same person can have a Liar in multiple events."""
    U = get_user_model()
    host = U.objects.create(username="host", email="host@x.com")
    user = U.objects.create(username="repeat", email="repeat@x.com")

    e1 = Event.objects.create(host=host, title="E1", start_at=timezone.now(), end_at=timezone.now())
    e2 = Event.objects.create(host=host, title="E2", start_at=timezone.now(), end_at=timezone.now())

    a1 = Attendee.objects.create(event=e1, user=user, rsvp_status=Attendee.RSVP.GOING)
    a2 = Attendee.objects.create(event=e2, user=user, rsvp_status=Attendee.RSVP.GOING)

    l1 = LiarProfile.objects.create(event=e1, inviter=a1, display_name="Liar-E1")
    l2 = LiarProfile.objects.create(event=e2, inviter=a2, display_name="Liar-E2")

    assert l1.event_id == e1.id
    assert l2.event_id == e2.id
    assert l1.inviter_id != l2.inviter_id  # different Attendee rows


@pytest.mark.django_db
def test_consent_timestamp_helper_sets_value():
    U = get_user_model()
    host = U.objects.create(username="h", email="h@x.com")
    e = Event.objects.create(host=host, title="T", start_at=timezone.now(), end_at=timezone.now())
    u = U.objects.create(username="u", email="u@x.com")
    a = Attendee.objects.create(event=e, user=u, rsvp_status=Attendee.RSVP.GOING)
    l = LiarProfile.objects.create(event=e, inviter=a, display_name="L")

    assert l.consent_at is None
    l.give_consent()
    assert l.consent_at is not None
```

