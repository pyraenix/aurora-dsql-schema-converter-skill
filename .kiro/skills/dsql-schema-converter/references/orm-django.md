# Django ORM Migration Guide for DSQL

How to run Django applications against Aurora DSQL.

Sources:
- [Aurora DSQL Django Adapter](https://github.com/awslabs/aurora-dsql-orms/tree/main/python/django)
- [aurora-dsql-django on PyPI](https://pypi.org/project/aurora-dsql-django/)
- [Django Pet Clinic Example](https://github.com/awslabs/aurora-dsql-orms/tree/main/python/django/examples/pet-clinic-app)
- [Aurora DSQL Connectivity Tools](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/aws-sdks.html)

---

## 1. Installation

```bash
pip install aurora-dsql-django boto3
```

The `aurora-dsql-django` adapter handles IAM token generation automatically for every connection.

---

## 2. Database Configuration

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'aurora_dsql_django',           # NOT 'django.db.backends.postgresql'
        'NAME': 'postgres',                        # Always 'postgres' for DSQL
        'HOST': '<cluster-id>.<region>.dsql.amazonaws.com',
        'PORT': '5432',
        'OPTIONS': {
            'sslmode': 'require',
        },
    }
}
```

**Key differences from standard PostgreSQL:**
- Engine is `aurora_dsql_django` (not `django.db.backends.postgresql`)
- No `USER` or `PASSWORD` — IAM token is generated automatically via boto3
- Database name is always `postgres`
- SSL is required

### AWS Credentials

The adapter uses boto3's credential chain. Ensure one of:
- Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- AWS credentials file (`~/.aws/credentials`)
- IAM role (EC2 instance profile, ECS task role, Lambda execution role)

---

## 3. Model Changes

### Disable Foreign Keys

Django auto-generates FK constraints. DSQL doesn't support them.

```python
# models.py

# BAD: Django will try to create FK constraint
class Ticket(models.Model):
    org = models.ForeignKey(Organization, on_delete=models.CASCADE)

# GOOD: Use integer/UUID field + application-layer validation
class Ticket(models.Model):
    org_id = models.BigIntegerField()  # No ForeignKey
    reporter_id = models.UUIDField()   # No ForeignKey

    def clean(self):
        """Application-layer FK validation."""
        if not Organization.objects.filter(id=self.org_id).exists():
            raise ValidationError({'org_id': 'Organization does not exist'})
        if not User.objects.filter(id=self.reporter_id).exists():
            raise ValidationError({'reporter_id': 'User does not exist'})
```

### Use UUID Primary Keys

```python
import uuid
from django.db import models

class BaseModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    class Meta:
        abstract = True

class Organization(BaseModel):
    name = models.CharField(max_length=200, unique=True)
    settings = models.JSONField(default=dict)  # Stored as json in DSQL
    created_at = models.DateTimeField(auto_now_add=True)
```

### JSONField Works Directly

Django's `JSONField` works with DSQL's `json` type:

```python
class UserProfile(BaseModel):
    preferences = models.JSONField(default=dict)  # ✅ Works
    # Queries with JSON lookups work:
    # UserProfile.objects.filter(preferences__theme='dark')
```

### Avoid Unsupported Field Types

| Django Field | DSQL Behavior | Alternative |
|---|---|---|
| `ForeignKey` | FK constraint fails | Use `BigIntegerField` / `UUIDField` |
| `ArrayField` | Not a stored type | Use `JSONField` with list |
| `HStoreField` | Not supported | Use `JSONField` |
| `SearchVectorField` | No FTS | Use external search (OpenSearch) |
| `CITextField` / `CIEmailField` | No citext extension | Use `CharField` + `lower()` queries |

---

## 4. Migrations

### Disable FK Constraints in Migrations

```python
# Custom migration operation to skip FK creation
from django.db import migrations

class Migration(migrations.Migration):
    operations = [
        migrations.CreateModel(
            name='Ticket',
            fields=[
                ('id', models.UUIDField(primary_key=True, default=uuid.uuid4)),
                # Use plain fields instead of ForeignKey
                ('org_id', models.BigIntegerField(db_index=True)),
                ('reporter_id', models.UUIDField(db_index=True)),
                ('title', models.CharField(max_length=500)),
            ],
        ),
    ]
```

### Run Migrations One at a Time

DSQL requires one DDL per transaction. Django migrations may batch DDL:

```bash
# Run migrations with --run-syncdb for initial setup
python manage.py migrate --run-syncdb

# If migrations fail, run them individually:
python manage.py migrate app_name 0001
python manage.py migrate app_name 0002
```

### Custom Migration Backend (if needed)

For complex migrations, wrap each operation in its own transaction:

```python
from django.db import migrations, connection

def run_ddl_separately(apps, schema_editor):
    with connection.cursor() as cursor:
        cursor.execute("CREATE INDEX ASYNC idx_tickets_org ON tickets (org_id)")

class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(run_ddl_separately),
    ]
```

---

## 5. OCC Retry Middleware

Every write can fail with serialization error. Add retry logic:

```python
# middleware/dsql_retry.py
import time
import random
from django.db import OperationalError, transaction

class DSQLRetryMiddleware:
    """Retry middleware for DSQL OCC conflicts (SQLSTATE 40001)."""

    def __init__(self, get_response):
        self.get_response = get_response
        self.max_retries = 5
        self.base_delay = 0.05  # 50ms

    def __call__(self, request):
        return self.get_response(request)

    def process_exception(self, request, exception):
        # Django wraps psycopg2 errors in OperationalError
        if hasattr(exception, '__cause__') and hasattr(exception.__cause__, 'pgcode'):
            if exception.__cause__.pgcode == '40001':
                return None  # Let view-level retry handle it
        return None


def with_occ_retry(func, max_retries=5):
    """Decorator for OCC retry on view functions or service methods."""
    def wrapper(*args, **kwargs):
        for attempt in range(max_retries):
            try:
                with transaction.atomic():
                    return func(*args, **kwargs)
            except OperationalError as e:
                if hasattr(e, '__cause__') and hasattr(e.__cause__, 'pgcode'):
                    if e.__cause__.pgcode == '40001' and attempt < max_retries - 1:
                        delay = min(0.05 * (2 ** attempt) + random.uniform(0, 0.05), 5.0)
                        time.sleep(delay)
                        continue
                raise
    return wrapper
```

Usage in views:

```python
from middleware.dsql_retry import with_occ_retry

@with_occ_retry
def create_ticket(org_id, reporter_id, title):
    ticket = Ticket(org_id=org_id, reporter_id=reporter_id, title=title)
    ticket.full_clean()  # Triggers FK validation in clean()
    ticket.save()
    return ticket
```

---

## 6. Signals for FK Validation

Use Django signals to enforce FK integrity:

```python
# signals.py
from django.db.models.signals import pre_save
from django.dispatch import receiver
from django.core.exceptions import ValidationError

@receiver(pre_save, sender=Ticket)
def validate_ticket_fks(sender, instance, **kwargs):
    if not Organization.objects.filter(id=instance.org_id).exists():
        raise ValidationError(f"Organization {instance.org_id} does not exist")
    if not User.objects.filter(id=instance.reporter_id).exists():
        raise ValidationError(f"Reporter {instance.reporter_id} does not exist")
```

---

## 7. Collation Considerations

Django's default `ORDER BY` will use C collation in DSQL:

```python
# This will sort by byte order (uppercase before lowercase)
Organization.objects.order_by('name')

# For case-insensitive ordering:
from django.db.models.functions import Lower
Organization.objects.order_by(Lower('name'))
```

---

## 8. Things to Remove from Django Settings

```python
# Remove or don't use:
# - django.contrib.postgres (ArrayField, HStoreField, etc.)
# - Any CONN_MAX_AGE > 3600 (DSQL connections timeout at 1 hour)
# - Database routers that assume multiple databases per cluster

# Set connection max age below DSQL's 1-hour timeout:
DATABASES['default']['CONN_MAX_AGE'] = 1800  # 30 minutes
```

---

## 9. Checklist

- [ ] Install `aurora-dsql-django` and `boto3`
- [ ] Change ENGINE to `aurora_dsql_django`
- [ ] Remove USER/PASSWORD from database config
- [ ] Replace all `ForeignKey` fields with plain ID fields
- [ ] Add `clean()` or signal-based FK validation
- [ ] Use `UUIDField` for primary keys
- [ ] Add OCC retry middleware/decorator
- [ ] Set `CONN_MAX_AGE` ≤ 1800
- [ ] Remove `django.contrib.postgres` if using unsupported fields
- [ ] Test migrations run one DDL at a time
- [ ] Test ORDER BY behavior with C collation
