## Industry Standards and Practices for Django Projects

This document outlines comprehensive, industry-standard guidelines and best practices for structuring, developing, testing, securing, and deploying Django applications. Use it as a reference when setting up or auditing your own projects.

---

### 1. Project Organization & Twelve-Factor Principles

* **Single Repository**: Maintain one codebase per application.
* **12-Factor App Compliance**:

  * **Codebase**: One codebase tracked in revision control.
  * **Config**: Store config in environment variables (`.env`), not in code.
  * **Dependencies**: Explicitly declare and isolate dependencies.
  * **Backing Services**: Treat external services (DB, cache) as attached resources.
  * **Build, Release, Run**: Strict separation of build and run stages.
  * **Processes**: Execute the app as one or more stateless processes.
  * **Port Binding**: Export services via port binding.
  * **Concurrency**: Scale out via the process model.
  * **Disposability**: Fast startup and graceful shutdown.
  * **Dev/Prod Parity**: Keep development, staging, and production as similar as possible.
  * **Logs**: Output logs as event streams.
  * **Admin Processes**: Run admin/management tasks as one-off processes.

### 2. Folder Structure

```
project_root/
├── apps/                     # First-party Django apps
├── config/                   # Project configuration
│   ├── asgi.py
│   ├── wsgi.py
│   ├── urls.py
│   └── settings/
│       ├── base.py
│       ├── dev.py
│       └── prod.py
├── requirements.txt
├── .env                      # Environment-specific secrets
├── Dockerfile                # Containerization
├── docker-compose.yml        # Local dev orchestration
├── manage.py
├── README.md
└── docs/                     # Architecture and onboarding docs
```

* **`apps/`**: Modular Django apps with clear boundaries.
* **`config/settings/`**: Split settings (common, development, production).
* **`.env`**: Environment variables for secrets (never commit).
* **`Dockerfile` / `docker-compose.yml`**: Infrastructure-as-code for reproducible environments.

---

### 3. Settings Management

* **Environment Variables**: Use `django-environ` or `python-decouple`.
* **Settings Split**:

  * **`base.py`**: Common settings (INSTALLED\_APPS, MIDDLEWARE, TEMPLATES, default DB).
  * **`dev.py`**: `DEBUG = True`, local DB, debug toolbar, verbose logging.
  * **`prod.py`**: `DEBUG = False`, security hardening, Sentry integration, whitenoise for static files.
* **Security Settings**:

  ```python
  SECURE_SSL_REDIRECT = True
  SESSION_COOKIE_SECURE = True
  CSRF_COOKIE_SECURE = True
  X_FRAME_OPTIONS = 'DENY'
  ```
* **Database Configuration**:

  * Use `DATABASE_URL` via env vars for portability.
  * Connection pooling with PgBouncer for PostgreSQL.

---

### 4. Virtual Environments & Dependency Management

* **Isolate Environments**: Use `venv`, `pipenv`, or `poetry`.
* **Pin Dependencies**: `requirements.txt` with exact versions.
* **Lockfiles**: Pipfile.lock or poetry.lock for reproducible installs.
* **Pre-commit Hooks**: `black`, `isort`, `flake8`, `mypy` for code quality.

---

### 5. App Structure & Registration

* **Creating an App**:

  ```bash
  python manage.py startapp users apps/users
  ```
* **App Directory Layout**:

  ```
  apps/users/
  ├── migrations/
  ├── __init__.py
  ├── admin.py
  ├── apps.py
  ├── models.py
  ├── views.py
  ├── serializers.py       # DRF
  ├── permissions.py       # DRF custom permissions
  ├── urls.py
  ├── tests.py
  └── templates/users/     # App-specific templates
  ```
* **Registering Apps** in `config/settings/base.py`:

  ```python
  INSTALLED_APPS += [
      'rest_framework',
      'apps.users.apps.UsersConfig',
      'apps.blog.apps.BlogConfig',
  ]
  ```

---

### 6. Models: Design & Best Practices

* **Meta Options**:

  ```python
  class Post(models.Model):
      title = models.CharField(max_length=200)
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ['-created_at']
          constraints = [
              models.UniqueConstraint(fields=['title'], name='unique_post_title')
          ]
  ```
* **Field Choices**: Use `choices` for enumerations.
* **Abstract/Base Models**: DRY for common fields:

  ```python
  class TimeStampedModel(models.Model):
      created_on = models.DateTimeField(auto_now_add=True)
      updated_on = models.DateTimeField(auto_now=True)

      class Meta:
          abstract = True
  ```
* **Validation**: Override `clean()` for cross-field logic.
* **String Representation**: Always define `__str__`.

---

### 7. Views: FBV vs CBV vs ViewSets

* **Function-Based Views (FBV)**: Simple endpoints.
* **Class-Based Views (CBV)**: Use Django generic views for common patterns.
* **Django REST Framework ViewSets**: For full CRUD REST APIs:

  ```python
  class UserViewSet(viewsets.ModelViewSet):
      queryset = User.objects.all()
      serializer_class = UserSerializer
  ```

---

### 8. URL Routing & Namespacing

* **Project URLs** (`config/urls.py`):

  ```python
  urlpatterns = [
      path('admin/', admin.site.urls),
      path('api/', include('apps.users.api_urls')),
      path('', include('apps.blog.urls', namespace='blog')),
  ]
  ```
* **App URLs** (`apps/users/urls.py`):

  ```python
  app_name = 'users'
  urlpatterns = [
      path('', user_list, name='list'),
  ]
  ```
* **API URLs** (`apps/users/api_urls.py`):

  ```python
  router = DefaultRouter()
  router.register(r'users', UserViewSet, basename='user')
  urlpatterns = router.urls
  ```

---

### 9. Templates & Static Files

* **Global Templates**: `/templates/base.html`.
* **App Templates**: `/apps/<app>/templates/<app>/*.html`.
* **Static Files**: `/static/css`, `/static/js`, `/static/images`.
* **Production**: Serve via `whitenoise` or CDN/S3.

---

### 10. API Design with Django REST Framework

* **Serializers**:

  ```python
  class UserSerializer(serializers.ModelSerializer):
      class Meta:
          model = User
          fields = ['id','username','email','date_joined']
  ```
* **Pagination & Filtering**:

  ```python
  REST_FRAMEWORK = {
      'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
      'PAGE_SIZE': 20,
      'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
  }
  ```
* **Authentication**: JWT (`simplejwt`), OAuth2 (`django-oauth-toolkit`).
* **Permissions**: Custom classes in `permissions.py`.

---

### 11. Testing Strategy

* **Unit Tests**: Models, utilities.
* **Integration Tests**: Views, APIs using `APIClient`.
* **Test Data**: Fixtures or `factory_boy`.
* **Coverage**: ≥ 90%.
* **CI**: Automate tests on Pull Requests (GitHub Actions).

---

### 12. Code Quality & Maintenance

* **Linters/Formatters**:

  * `black`, `isort` for formatting
  * `flake8` for style
  * `mypy` for type checking
* **Pre-commit Hooks**: Enforce checks before commits.
* **Documentation**:

  * `docs/` with architecture overviews.
  * Auto-generated API docs with Swagger/OpenAPI (`drf-yasg`).

---

### 13. Security & Compliance

* **Dependency Scanning**: `pip-audit`, `safety`.
* **HTTPS**: Enforce SSL with HSTS.
* **CSRF**: Enabled for all state-changing views.
* **Rate Limiting**: via `django-ratelimit` or DRF throttling.
* **Audit Logs**: Track sensitive actions.

---

### 14. Deployment & DevOps

1. **Containerization**:

   * Multi-stage `Dockerfile` for lean images.
   * `docker-compose` for local dev with Postgres, Redis, Celery.
2. **Process Management**:

   * Gunicorn behind Nginx.
   * Supervisor or systemd for service management.
3. **CI/CD Pipeline**:

   * Test, lint, security scans on PRs.
   * Auto-deploy on merge to `main`.
4. **Monitoring & Logging**:

   * Centralized logs (ELK, Papertrail).
   * Error tracking (Sentry).
   * Health checks and uptime alerts.

---

### 15. Performance & Scaling

* **Database**: Connection pooling with PgBouncer.
* **Caching**: Redis for per-view and template fragment caching.
* **Async Tasks**: Celery + RabbitMQ/Redis.
* **CDN**: For static and media assets.
* **ORM Optimization**: `select_related()`, `prefetch_related()`, avoid N+1 queries.

---

## Conclusion

Adhering to these standards ensures your Django project is:

* **Maintainable**: Modular, well-organized code.
* **Secure**: Hardened settings, regular auditing.
* **Tested**: High coverage, CI enforcement.
* **Scalable**: Prepared for growth with caching, async tasks, containerization.

Use this document as a living guide—adapt tools and modules (GraphQL, Channels, etc.) as needed, but maintain the core structure and practices across all projects.


