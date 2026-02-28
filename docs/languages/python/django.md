# Django — Guia completo do básico ao avançado

Django é um framework web de alto nível para Python que incentiva o desenvolvimento rápido e o design limpo e pragmático. Segue o padrão **MVT** (Model-View-Template), inclui ORM próprio, sistema de admin, autenticação, e muito mais — "batteries included".

---

## 1. Instalação e criação de projeto

### Instalação

```bash
# Ambiente virtual (recomendado)
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
.venv\Scripts\Activate.ps1       # Windows PowerShell

pip install django
```

### Criar projeto e app

```bash
django-admin startproject myproject .
python manage.py startapp core
```

> **Dica:** o ponto `.` no final de `startproject` cria o projeto no diretório atual, evitando uma pasta aninhada extra.

### Primeiro run

```bash
python manage.py migrate          # aplica migrations iniciais (auth, sessions, etc.)
python manage.py runserver 0.0.0.0:8000
```

---

## 2. Estrutura do projeto

```
myproject/
├── manage.py                  # CLI do projeto
├── myproject/
│   ├── __init__.py
│   ├── asgi.py                # entrypoint ASGI
│   ├── settings.py            # configurações
│   ├── urls.py                # roteamento raiz
│   └── wsgi.py                # entrypoint WSGI
└── core/                      # app exemplo
    ├── __init__.py
    ├── admin.py               # registro no admin
    ├── apps.py                # config da app
    ├── migrations/            # migrations do banco
    ├── models.py              # modelos (ORM)
    ├── tests.py               # testes
    ├── urls.py                # rotas da app (criar manualmente)
    └── views.py               # views
```

Registre a app em `INSTALLED_APPS`:

```python
# settings.py
INSTALLED_APPS = [
    # apps do Django
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # apps próprias
    "core",
]
```

---

## 3. Configurações (`settings.py`)

### Variáveis de ambiente com `django-environ`

```bash
pip install django-environ
```

```python
# settings.py
import environ

env = environ.Env(DEBUG=(bool, False))
environ.Env.read_env(".env")      # lê arquivo .env na raiz

SECRET_KEY = env("SECRET_KEY")
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=["localhost"])

DATABASES = {
    "default": env.db(),          # lê DATABASE_URL
}
```

```ini
# .env
SECRET_KEY=minha-chave-super-secreta
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
```

### Separar settings por ambiente

```
myproject/
  settings/
    __init__.py
    base.py          # configurações comuns
    development.py   # import de base + overrides
    production.py    # import de base + overrides
    testing.py       # import de base + overrides
```

```python
# settings/development.py
from .base import *  # noqa

DEBUG = True
ALLOWED_HOSTS = ["*"]
```

Selecione o módulo via variável de ambiente:

```bash
export DJANGO_SETTINGS_MODULE=myproject.settings.production
```

### Configurações importantes para produção

```python
DEBUG = False
ALLOWED_HOSTS = ["meudominio.com", "www.meudominio.com"]
SECRET_KEY = env("SECRET_KEY")   # nunca comite no repositório

# Segurança HTTP
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = "DENY"

# Arquivos estáticos
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

---

## 4. Models e ORM

### Definindo modelos

```python
# core/models.py
from django.db import models
from django.conf import settings


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)

    class Meta:
        verbose_name_plural = "categories"
        ordering = ["name"]

    def __str__(self):
        return self.name


class Article(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Rascunho"
        PUBLISHED = "published", "Publicado"

    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    body = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="articles",
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="articles",
    )
    tags = models.ManyToManyField("Tag", blank=True, related_name="articles")
    status = models.CharField(
        max_length=10,
        choices=Status.choices,
        default=Status.DRAFT,
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["-created_at"]),
            models.Index(fields=["slug"]),
        ]

    def __str__(self):
        return self.title


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name
```

### Tipos de campos mais comuns

| Campo | Descrição |
|---|---|
| `CharField(max_length=N)` | Texto curto |
| `TextField()` | Texto longo |
| `IntegerField()` | Inteiro |
| `FloatField()` | Ponto flutuante |
| `DecimalField(max_digits, decimal_places)` | Decimal preciso |
| `BooleanField(default=False)` | Booleano |
| `DateField()` / `DateTimeField()` | Data / Data+hora |
| `EmailField()` | E-mail (com validação) |
| `URLField()` | URL |
| `SlugField()` | Slug (usado em URLs) |
| `FileField(upload_to="...")` | Upload de arquivo |
| `ImageField(upload_to="...")` | Upload de imagem |
| `JSONField()` | Dado JSON (Postgres nativo, SQLite 3.9+) |
| `UUIDField()` | UUID |
| `ForeignKey(Model, on_delete=...)` | Relação N:1 |
| `ManyToManyField(Model)` | Relação N:N |
| `OneToOneField(Model, on_delete=...)` | Relação 1:1 |

### Opções de `on_delete`

| Opção | Comportamento |
|---|---|
| `CASCADE` | Apaga os filhos junto |
| `PROTECT` | Impede exclusão se houver referência |
| `SET_NULL` | Define como `NULL` (precisa de `null=True`) |
| `SET_DEFAULT` | Define o valor default |
| `DO_NOTHING` | Não faz nada (cuidado com integridade) |

### QuerySet API — consultas ao banco

```python
from core.models import Article

# Criar
article = Article.objects.create(title="Título", slug="titulo", body="Corpo", author=user)

# Ler
Article.objects.all()                              # todos
Article.objects.filter(status="published")         # filtrar
Article.objects.exclude(status="draft")            # excluir condição
Article.objects.get(pk=1)                          # um único objeto (ou DoesNotExist)
Article.objects.first()                            # primeiro resultado
Article.objects.last()                             # último resultado
Article.objects.count()                            # contagem

# Filtros avançados
Article.objects.filter(title__icontains="django")  # LIKE case-insensitive
Article.objects.filter(created_at__year=2025)      # por ano
Article.objects.filter(created_at__gte="2025-01-01")  # >= data
Article.objects.filter(category__name="Python")    # lookup em FK
Article.objects.filter(tags__name__in=["django", "python"])  # M2M lookup

# Encadeamento
Article.objects.filter(status="published").order_by("-created_at")[:10]

# Atualizar em massa
Article.objects.filter(status="draft").update(status="published")

# Deletar
Article.objects.filter(created_at__lt="2020-01-01").delete()
```

### Objetos Q — consultas complexas (AND, OR, NOT)

```python
from django.db.models import Q

# OR
Article.objects.filter(Q(status="published") | Q(author=user))

# AND explícito
Article.objects.filter(Q(status="published") & Q(category__name="Python"))

# NOT
Article.objects.filter(~Q(status="draft"))

# Combinações
Article.objects.filter(
    (Q(title__icontains="django") | Q(body__icontains="django"))
    & Q(status="published")
)
```

### Objetos F — referência a campos do próprio modelo

```python
from django.db.models import F

# Comparar campos entre si
Article.objects.filter(updated_at__gt=F("created_at"))

# Atualizar com base no valor atual
Article.objects.filter(pk=1).update(views=F("views") + 1)
```

### Agregações e anotações

```python
from django.db.models import Count, Avg, Sum, Max, Min

# Agregar
Article.objects.aggregate(total=Count("id"), avg_views=Avg("views"))
# => {"total": 50, "avg_views": 120.5}

# Anotar (adicionar campo calculado a cada objeto)
categories = Category.objects.annotate(
    num_articles=Count("articles"),
).order_by("-num_articles")
for cat in categories:
    print(cat.name, cat.num_articles)
```

### Otimizações de QuerySet

```python
# select_related: JOIN para FK e OneToOne (evita N+1)
articles = Article.objects.select_related("author", "category").all()

# prefetch_related: query separada para M2M e FK reverso
articles = Article.objects.prefetch_related("tags").all()

# Prefetch customizado
from django.db.models import Prefetch
articles = Article.objects.prefetch_related(
    Prefetch("tags", queryset=Tag.objects.filter(name__startswith="py"))
)

# values / values_list (retorna dicts / tuplas, sem instanciar model)
Article.objects.values("title", "author__username")
Article.objects.values_list("title", flat=True)

# only / defer (carregamento parcial de colunas)
Article.objects.only("title", "slug")    # carrega só esses campos
Article.objects.defer("body")            # carrega tudo EXCETO body

# exists() (mais eficiente que count() > 0)
Article.objects.filter(status="published").exists()
```

### Model Managers personalizados

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="published")


class Article(models.Model):
    # ...
    objects = models.Manager()      # manager padrão
    published = PublishedManager()   # manager custom

# Uso
Article.published.all()   # retorna apenas publicados
```

### Propriedades e métodos no model

```python
class Article(models.Model):
    # ... campos ...

    @property
    def is_recent(self):
        from django.utils import timezone
        return self.created_at >= timezone.now() - timezone.timedelta(days=7)

    def get_absolute_url(self):
        from django.urls import reverse
        return reverse("article-detail", kwargs={"slug": self.slug})
```

### Abstract models e mixins

```python
class TimestampMixin(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Article(TimestampMixin, models.Model):
    title = models.CharField(max_length=200)
    # ... herda created_at e updated_at automaticamente
```

---

## 5. Migrations

```bash
# Gerar migrations após alterar models
python manage.py makemigrations

# Gerar migration para app específica
python manage.py makemigrations core

# Aplicar migrations
python manage.py migrate

# Aplicar até uma migration específica (rollback parcial)
python manage.py migrate core 0003_auto

# Reverter todas as migrations de uma app
python manage.py migrate core zero

# Ver status das migrations
python manage.py showmigrations

# Ver SQL que uma migration vai executar
python manage.py sqlmigrate core 0001_initial
```

### Migration de dados (data migration)

```bash
python manage.py makemigrations core --empty --name populate_categories
```

```python
# core/migrations/0002_populate_categories.py
from django.db import migrations


def create_categories(apps, schema_editor):
    Category = apps.get_model("core", "Category")
    for name in ["Python", "Django", "DevOps"]:
        Category.objects.get_or_create(name=name, slug=name.lower())


def reverse_categories(apps, schema_editor):
    Category = apps.get_model("core", "Category")
    Category.objects.filter(slug__in=["python", "django", "devops"]).delete()


class Migration(migrations.Migration):
    dependencies = [
        ("core", "0001_initial"),
    ]
    operations = [
        migrations.RunPython(create_categories, reverse_categories),
    ]
```

### Squash de migrations

```bash
# Combinar migrations 0001 até 0010 em uma só
python manage.py squashmigrations core 0001 0010
```

---

## 6. URLs e roteamento

### URL raiz do projeto

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("core.urls", namespace="core")),
]

# Servir media em desenvolvimento
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### URLs da app

```python
# core/urls.py
from django.urls import path
from . import views

app_name = "core"

urlpatterns = [
    path("", views.ArticleListView.as_view(), name="article-list"),
    path("<slug:slug>/", views.ArticleDetailView.as_view(), name="article-detail"),
    path("category/<slug:slug>/", views.category_articles, name="category-articles"),
    path("search/", views.search, name="search"),
]
```

### Converters de path

| Converter | Exemplo | Descrição |
|---|---|---|
| `<int:pk>` | `42` | Inteiro |
| `<str:name>` | `hello` | String sem `/` |
| `<slug:slug>` | `meu-artigo` | Slug |
| `<uuid:id>` | `075194d3-...` | UUID |
| `<path:rest>` | `foo/bar/baz` | Caminho completo (com `/`) |

### Converter customizado

```python
# core/converters.py
class FourDigitYearConverter:
    regex = r"[0-9]{4}"

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return f"{value:04d}"
```

```python
# core/urls.py
from django.urls import path, register_converter
from .converters import FourDigitYearConverter

register_converter(FourDigitYearConverter, "yyyy")

urlpatterns = [
    path("articles/<yyyy:year>/", views.articles_by_year, name="articles-year"),
]
```

### `reverse()` e `{% url %}` — gerar URLs

```python
from django.urls import reverse

url = reverse("core:article-detail", kwargs={"slug": "meu-artigo"})
# => "/api/meu-artigo/"
```

```html
<!-- Em templates -->
<a href="{% url 'core:article-detail' slug=article.slug %}">{{ article.title }}</a>
```

---

## 7. Views

### Function-Based Views (FBV)

```python
# core/views.py
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse, Http404
from .models import Article, Category


def article_list(request):
    articles = Article.published.select_related("author", "category")
    return render(request, "core/article_list.html", {"articles": articles})


def article_detail(request, slug):
    article = get_object_or_404(Article.published, slug=slug)
    return render(request, "core/article_detail.html", {"article": article})


def category_articles(request, slug):
    category = get_object_or_404(Category, slug=slug)
    articles = category.articles.filter(status="published")
    return render(request, "core/category.html", {
        "category": category,
        "articles": articles,
    })
```

### Class-Based Views (CBV)

```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.urls import reverse_lazy
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from .models import Article
from .forms import ArticleForm


class ArticleListView(ListView):
    model = Article
    template_name = "core/article_list.html"
    context_object_name = "articles"
    paginate_by = 20
    queryset = Article.published.select_related("author", "category")


class ArticleDetailView(DetailView):
    model = Article
    template_name = "core/article_detail.html"
    slug_field = "slug"
    slug_url_kwarg = "slug"


class ArticleCreateView(LoginRequiredMixin, CreateView):
    model = Article
    form_class = ArticleForm
    template_name = "core/article_form.html"
    success_url = reverse_lazy("core:article-list")

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class ArticleUpdateView(LoginRequiredMixin, UpdateView):
    model = Article
    form_class = ArticleForm
    template_name = "core/article_form.html"

    def get_queryset(self):
        # só permite editar artigos do próprio autor
        return super().get_queryset().filter(author=self.request.user)


class ArticleDeleteView(LoginRequiredMixin, DeleteView):
    model = Article
    success_url = reverse_lazy("core:article-list")

    def get_queryset(self):
        return super().get_queryset().filter(author=self.request.user)
```

### Hierarquia das CBVs principais

```
View
├── TemplateView
├── RedirectView
├── ListView
├── DetailView
├── FormView
├── CreateView
├── UpdateView
└── DeleteView
```

> **Dica:** consulte [ccbv.co.uk](https://ccbv.co.uk/) para ver a árvore de herança completa das CBVs.

### Decorators úteis em FBVs

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_http_methods
from django.views.decorators.cache import cache_page

@login_required
@require_http_methods(["GET", "POST"])
def create_article(request):
    # ...
    pass

@cache_page(60 * 15)  # cache de 15 minutos
def article_list(request):
    # ...
    pass
```

### Retornando JSON (API simples)

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json

@csrf_exempt
def api_articles(request):
    if request.method == "GET":
        articles = list(
            Article.published.values("id", "title", "slug", "created_at")
        )
        return JsonResponse({"articles": articles}, safe=False)

    if request.method == "POST":
        data = json.loads(request.body)
        # ... processar ...
        return JsonResponse({"status": "created"}, status=201)
```

---

## 8. Templates

### Configuração

```python
# settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],   # templates globais
        "APP_DIRS": True,                     # templates em app/templates/
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

### Template base + herança

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Meu Site{% endblock %}</title>
    {% block extra_css %}{% endblock %}
</head>
<body>
    <nav>
        <a href="{% url 'core:article-list' %}">Artigos</a>
        {% if user.is_authenticated %}
            <span>{{ user.username }}</span>
            <a href="{% url 'logout' %}">Sair</a>
        {% else %}
            <a href="{% url 'login' %}">Entrar</a>
        {% endif %}
    </nav>

    <main>
        {% block content %}{% endblock %}
    </main>

    {% block extra_js %}{% endblock %}
</body>
</html>
```

```html
<!-- templates/core/article_list.html -->
{% extends "base.html" %}

{% block title %}Artigos{% endblock %}

{% block content %}
<h1>Artigos</h1>
{% for article in articles %}
    <article>
        <h2><a href="{{ article.get_absolute_url }}">{{ article.title }}</a></h2>
        <p>{{ article.body|truncatewords:30 }}</p>
        <small>{{ article.created_at|date:"d/m/Y H:i" }} — {{ article.author }}</small>
    </article>
{% empty %}
    <p>Nenhum artigo encontrado.</p>
{% endfor %}

<!-- Paginação -->
{% if is_paginated %}
<nav>
    {% if page_obj.has_previous %}
        <a href="?page={{ page_obj.previous_page_number }}">Anterior</a>
    {% endif %}
    Página {{ page_obj.number }} de {{ page_obj.paginator.num_pages }}
    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">Próxima</a>
    {% endif %}
</nav>
{% endif %}
{% endblock %}
```

### Tags e filtros úteis nos templates

```html
<!-- Condicionais -->
{% if article.is_recent %}<span>Novo!</span>{% endif %}

<!-- Loop com forloop -->
{% for tag in article.tags.all %}
    {{ tag.name }}{% if not forloop.last %}, {% endif %}
{% endfor %}

<!-- Include (componentes reutilizáveis) -->
{% include "partials/_article_card.html" with article=article %}

<!-- Filtros comuns -->
{{ text|linebreaksbr }}
{{ text|safe }}            {# marca como seguro (cuidado com XSS) #}
{{ list|length }}
{{ price|floatformat:2 }}  {# 19.90 #}
{{ name|default:"Anônimo" }}
{{ html_content|striptags }}
{{ text|slugify }}
{{ date|timesince }}       {# "há 3 dias" #}
```

### Template tags customizadas

```python
# core/templatetags/core_tags.py
from django import template
from core.models import Category

register = template.Library()

@register.simple_tag
def get_categories():
    return Category.objects.all()

@register.inclusion_tag("partials/_categories.html")
def show_categories():
    return {"categories": Category.objects.all()}

@register.filter
def word_count(value):
    return len(value.split())
```

```html
{% load core_tags %}
{% get_categories as categories %}
{% show_categories %}
{{ article.body|word_count }}
```

---

## 9. Forms

### Django Forms

```python
# core/forms.py
from django import forms
from .models import Article


# Form independente do model
class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, label="Nome")
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)

    def clean_email(self):
        email = self.cleaned_data["email"]
        if not email.endswith("@exemplo.com"):
            raise forms.ValidationError("Apenas e-mails corporativos.")
        return email


# ModelForm — gerado a partir do model
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "slug", "body", "category", "tags", "status"]
        widgets = {
            "body": forms.Textarea(attrs={"rows": 10}),
            "tags": forms.CheckboxSelectMultiple,
        }
        labels = {
            "title": "Título",
            "body": "Conteúdo",
        }

    def clean_title(self):
        title = self.cleaned_data["title"]
        if len(title) < 5:
            raise forms.ValidationError("O título deve ter pelo menos 5 caracteres.")
        return title
```

### Usando o form na view

```python
def contact(request):
    if request.method == "POST":
        form = ContactForm(request.POST)
        if form.is_valid():
            # form.cleaned_data contém os dados validados
            send_email(form.cleaned_data)
            return redirect("core:success")
    else:
        form = ContactForm()
    return render(request, "core/contact.html", {"form": form})
```

### Renderização no template

```html
<form method="post">
    {% csrf_token %}

    <!-- Renderização automática -->
    {{ form.as_p }}

    <!-- Ou campo a campo (controle total do HTML) -->
    {% for field in form %}
    <div class="field {% if field.errors %}has-error{% endif %}">
        <label for="{{ field.id_for_label }}">{{ field.label }}</label>
        {{ field }}
        {% for error in field.errors %}
            <span class="error">{{ error }}</span>
        {% endfor %}
    </div>
    {% endfor %}

    <button type="submit">Enviar</button>
</form>
```

### Formsets

```python
from django.forms import modelformset_factory

ArticleFormSet = modelformset_factory(
    Article,
    fields=["title", "status"],
    extra=2,     # 2 forms em branco adicionais
    max_num=10,
)

def manage_articles(request):
    formset = ArticleFormSet(request.POST or None, queryset=Article.objects.filter(author=request.user))
    if formset.is_valid():
        formset.save()
        return redirect("core:article-list")
    return render(request, "core/manage_articles.html", {"formset": formset})
```

---

## 10. Django Admin

### Registro básico

```python
# core/admin.py
from django.contrib import admin
from .models import Article, Category, Tag


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ["name", "slug"]
    prepopulated_fields = {"slug": ("name",)}


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ["name"]


@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "category", "status", "created_at"]
    list_filter = ["status", "category", "created_at"]
    search_fields = ["title", "body"]
    prepopulated_fields = {"slug": ("title",)}
    raw_id_fields = ["author"]             # widget de busca para FK com muitos registros
    date_hierarchy = "created_at"
    list_editable = ["status"]             # edição inline na listagem
    readonly_fields = ["created_at", "updated_at"]
    list_per_page = 25

    fieldsets = (
        (None, {
            "fields": ("title", "slug", "body"),
        }),
        ("Classificação", {
            "fields": ("category", "tags", "status"),
        }),
        ("Metadados", {
            "classes": ("collapse",),       # seção recolhível
            "fields": ("author", "created_at", "updated_at"),
        }),
    )

    # Actions customizadas
    @admin.action(description="Marcar selecionados como publicados")
    def make_published(self, request, queryset):
        queryset.update(status="published")

    actions = ["make_published"]
```

### Inline (editar modelos relacionados)

```python
class ArticleInline(admin.TabularInline):    # ou StackedInline
    model = Article
    extra = 0
    fields = ["title", "status"]
    readonly_fields = ["created_at"]

@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    inlines = [ArticleInline]
```

### Customizar o título do Admin

```python
# myproject/urls.py ou admin.py
admin.site.site_header = "Meu Painel Admin"
admin.site.site_title = "Admin"
admin.site.index_title = "Painel de Controle"
```

---

## 11. Autenticação e permissões

### Sistema built-in

```python
# settings.py
LOGIN_URL = "/accounts/login/"
LOGIN_REDIRECT_URL = "/"
LOGOUT_REDIRECT_URL = "/"

# Incluir URLs de autenticação do Django
# urls.py
from django.contrib.auth import views as auth_views

urlpatterns = [
    path("accounts/login/", auth_views.LoginView.as_view(), name="login"),
    path("accounts/logout/", auth_views.LogoutView.as_view(), name="logout"),
    path("accounts/password_change/", auth_views.PasswordChangeView.as_view(), name="password_change"),
    path("accounts/password_change/done/", auth_views.PasswordChangeDoneView.as_view(), name="password_change_done"),
]
```

### Proteger views

```python
# FBV
from django.contrib.auth.decorators import login_required, permission_required

@login_required
def minha_view(request):
    pass

@permission_required("core.add_article", raise_exception=True)
def criar_artigo(request):
    pass

# CBV
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin

class MinhaView(LoginRequiredMixin, ListView):
    pass

class CriarArtigo(PermissionRequiredMixin, CreateView):
    permission_required = "core.add_article"
```

### User model customizado

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True)
    birth_date = models.DateField(null=True, blank=True)

    def __str__(self):
        return self.username
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.User"   # DEFINA ANTES da primeira migration!
```

> **Importante:** defina `AUTH_USER_MODEL` **antes** de rodar a primeira `migrate`. Mudar depois é complexo.

### Criar usuários programaticamente

```python
from django.contrib.auth import get_user_model

User = get_user_model()

# Usuário normal
user = User.objects.create_user("joao", "joao@email.com", "senha123")

# Superusuário
admin = User.objects.create_superuser("admin", "admin@email.com", "admin123")
```

### Groups e Permissions

```python
from django.contrib.auth.models import Group, Permission
from django.contrib.contenttypes.models import ContentType
from core.models import Article

# Criar permissão customizada
content_type = ContentType.objects.get_for_model(Article)
perm = Permission.objects.create(
    codename="can_publish_article",
    name="Pode publicar artigos",
    content_type=content_type,
)

# Criar grupo e atribuir permissão
editors = Group.objects.create(name="Editores")
editors.permissions.add(perm)

# Adicionar usuário ao grupo
user.groups.add(editors)

# Verificar
user.has_perm("core.can_publish_article")  # True
```

---

## 12. Django REST Framework (DRF)

### Instalação

```bash
pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "rest_framework",
]

REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/hour",
        "user": "1000/hour",
    },
}
```

### Serializers

```python
# core/serializers.py
from rest_framework import serializers
from .models import Article, Category, Tag


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ["id", "name"]


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ["id", "name", "slug"]


class ArticleSerializer(serializers.ModelSerializer):
    author = serializers.StringRelatedField(read_only=True)
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(), source="category", write_only=True,
    )
    tags = TagSerializer(many=True, read_only=True)
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(), source="tags", many=True, write_only=True,
    )

    class Meta:
        model = Article
        fields = [
            "id", "title", "slug", "body", "author",
            "category", "category_id", "tags", "tag_ids",
            "status", "created_at", "updated_at",
        ]
        read_only_fields = ["created_at", "updated_at"]

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Título muito curto.")
        return value
```

### ViewSets + Router

```python
# core/api_views.py
from rest_framework import viewsets, permissions, filters
from django_filters.rest_framework import DjangoFilterBackend
from .models import Article
from .serializers import ArticleSerializer


class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.select_related("author", "category").prefetch_related("tags")
    serializer_class = ArticleSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ["status", "category"]
    search_fields = ["title", "body"]
    ordering_fields = ["created_at", "title"]
    ordering = ["-created_at"]
    lookup_field = "slug"

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

```python
# core/urls.py (API)
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .api_views import ArticleViewSet

router = DefaultRouter()
router.register("articles", ArticleViewSet, basename="article")

urlpatterns = [
    path("", include(router.urls)),
]
```

### Permissions customizadas

```python
# core/permissions.py
from rest_framework import permissions


class IsAuthorOrReadOnly(permissions.BasePermission):
    """Permite edição apenas ao autor do objeto."""

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user
```

### Autenticação via Token (JWT)

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=30),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
]
```

---

## 13. Middleware

### Middleware customizado

```python
# core/middleware.py
import time
import logging

logger = logging.getLogger(__name__)


class RequestTimingMiddleware:
    """Loga o tempo de processamento de cada request."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = time.monotonic() - start
        logger.info(f"{request.method} {request.path} — {duration:.3f}s")
        return response


class ForceJsonContentTypeMiddleware:
    """Força Content-Type JSON em rotas da API."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        if request.path.startswith("/api/"):
            response["Content-Type"] = "application/json"
        return response
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    # customizados
    "core.middleware.RequestTimingMiddleware",
]
```

> **Ordem importa!** O Django processa middlewares de cima para baixo no request e de baixo para cima no response.

---

## 14. Signals

Signals permitem desacoplar ações em resposta a eventos do Django.

```python
# core/signals.py
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver
from django.core.cache import cache
from .models import Article


@receiver(post_save, sender=Article)
def invalidate_cache_on_save(sender, instance, created, **kwargs):
    cache.delete(f"article_{instance.slug}")
    cache.delete("article_list")


@receiver(pre_delete, sender=Article)
def log_article_deletion(sender, instance, **kwargs):
    import logging
    logger = logging.getLogger(__name__)
    logger.warning(f"Artigo deletado: {instance.title} (pk={instance.pk})")
```

```python
# core/apps.py
from django.apps import AppConfig


class CoreConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "core"

    def ready(self):
        import core.signals  # noqa: F401 — registra os signals
```

### Signals built-in úteis

| Signal | Quando dispara |
|---|---|
| `pre_save` / `post_save` | Antes/depois de `model.save()` |
| `pre_delete` / `post_delete` | Antes/depois de `model.delete()` |
| `m2m_changed` | Alteração em campo ManyToMany |
| `request_started` / `request_finished` | Início/fim de cada request HTTP |
| `user_logged_in` / `user_logged_out` | Login/logout de usuário |

---

## 15. Management Commands customizados

```python
# core/management/commands/clean_old_articles.py
from django.core.management.base import BaseCommand
from django.utils import timezone
from core.models import Article


class Command(BaseCommand):
    help = "Remove artigos em rascunho com mais de 90 dias"

    def add_arguments(self, parser):
        parser.add_argument(
            "--days",
            type=int,
            default=90,
            help="Número de dias (padrão: 90)",
        )
        parser.add_argument(
            "--dry-run",
            action="store_true",
            help="Apenas mostra o que seria apagado, sem apagar de fato",
        )

    def handle(self, *args, **options):
        cutoff = timezone.now() - timezone.timedelta(days=options["days"])
        old_drafts = Article.objects.filter(status="draft", created_at__lt=cutoff)
        count = old_drafts.count()

        if options["dry_run"]:
            self.stdout.write(f"[DRY RUN] {count} artigos seriam removidos.")
            return

        old_drafts.delete()
        self.stdout.write(self.style.SUCCESS(f"{count} artigos removidos com sucesso."))
```

```bash
python manage.py clean_old_articles --days 60 --dry-run
```

---

## 16. Arquivos estáticos e media

### Configuração

```python
# settings.py
STATIC_URL = "/static/"
STATICFILES_DIRS = [BASE_DIR / "static"]   # arquivos durante desenvolvimento
STATIC_ROOT = BASE_DIR / "staticfiles"     # destino do collectstatic

MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

### Uso em templates

```html
{% load static %}
<link rel="stylesheet" href="{% static 'css/style.css' %}">
<img src="{% static 'images/logo.png' %}" alt="Logo">
```

### `collectstatic` para produção

```bash
python manage.py collectstatic --noinput
```

### WhiteNoise — servir estáticos sem Nginx

```bash
pip install whitenoise
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",    # logo após SecurityMiddleware
    # ...
]

STORAGES = {
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

---

## 17. Caching

### Backends de cache

```python
# settings.py

# Redis (recomendado para produção)
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
    }
}

# Memcached
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": "127.0.0.1:11211",
    }
}

# Banco de dados (para desenvolvimento)
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.db.DatabaseCache",
        "LOCATION": "cache_table",
    }
}
```

### Uso da cache API

```python
from django.core.cache import cache

# Set / Get
cache.set("minha_chave", "valor", timeout=300)     # 5 minutos
valor = cache.get("minha_chave", default="fallback")

# Get or set (atômico)
valor = cache.get_or_set("minha_chave", calcular_valor, timeout=600)

# Deletar
cache.delete("minha_chave")

# Incrementar / decrementar
cache.set("visits", 0)
cache.incr("visits")

# Múltiplas chaves
cache.set_many({"a": 1, "b": 2, "c": 3})
cache.get_many(["a", "b", "c"])
cache.delete_many(["a", "b", "c"])
```

### Cache em views

```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

# FBV
@cache_page(60 * 15)   # 15 minutos
def article_list(request):
    # ...

# CBV
@method_decorator(cache_page(60 * 15), name="dispatch")
class ArticleListView(ListView):
    # ...
```

### Cache de template fragments

```html
{% load cache %}
{% cache 600 sidebar request.user.id %}
    <!-- Conteúdo caro computacionalmente -->
    {% for cat in categories %}
        <a href="...">{{ cat.name }} ({{ cat.num_articles }})</a>
    {% endfor %}
{% endcache %}
```

---

## 18. Logging

```python
# settings.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
        "file": {
            "class": "logging.FileHandler",
            "filename": BASE_DIR / "logs" / "django.log",
            "formatter": "verbose",
        },
    },
    "loggers": {
        "django": {
            "handlers": ["console", "file"],
            "level": "WARNING",
        },
        "core": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}
```

```python
# Em qualquer lugar do código
import logging

logger = logging.getLogger(__name__)

logger.debug("Mensagem de debug")
logger.info("Informação")
logger.warning("Atenção")
logger.error("Erro", exc_info=True)     # inclui traceback
logger.critical("Erro crítico")
```

---

## 19. Testes

### Estrutura recomendada

```
core/
  tests/
    __init__.py
    test_models.py
    test_views.py
    test_forms.py
    test_api.py
    factories.py       # factories (factory_boy)
    conftest.py        # fixtures pytest
```

### TestCase do Django

```python
# core/tests/test_models.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from core.models import Article, Category

User = get_user_model()


class ArticleModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user("test", "test@test.com", "pass123")
        cls.category = Category.objects.create(name="Python", slug="python")
        cls.article = Article.objects.create(
            title="Test Article",
            slug="test-article",
            body="Body text",
            author=cls.user,
            category=cls.category,
            status="published",
        )

    def test_str(self):
        self.assertEqual(str(self.article), "Test Article")

    def test_get_absolute_url(self):
        self.assertEqual(self.article.get_absolute_url(), "/api/test-article/")

    def test_published_manager(self):
        self.assertIn(self.article, Article.published.all())
```

### Testes de views

```python
# core/tests/test_views.py
from django.test import TestCase, Client
from django.urls import reverse


class ArticleViewTests(TestCase):
    def setUp(self):
        self.client = Client()

    def test_article_list_status_200(self):
        response = self.client.get(reverse("core:article-list"))
        self.assertEqual(response.status_code, 200)

    def test_article_list_uses_correct_template(self):
        response = self.client.get(reverse("core:article-list"))
        self.assertTemplateUsed(response, "core/article_list.html")

    def test_article_create_requires_login(self):
        response = self.client.get(reverse("core:article-create"))
        self.assertEqual(response.status_code, 302)  # redirect ao login
```

### Testes de API (DRF)

```python
# core/tests/test_api.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth import get_user_model

User = get_user_model()


class ArticleAPITest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user("api_user", password="pass123")
        self.client = APIClient()

    def test_list_articles_unauthenticated(self):
        response = self.client.get("/api/articles/")
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_create_article_authenticated(self):
        self.client.force_authenticate(user=self.user)
        data = {
            "title": "Novo Artigo",
            "slug": "novo-artigo",
            "body": "Conteúdo do artigo",
            "status": "draft",
        }
        response = self.client.post("/api/articles/", data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    def test_create_article_unauthenticated(self):
        response = self.client.post("/api/articles/", {"title": "X"})
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

### Usando pytest + factory_boy

```bash
pip install pytest-django factory-boy
```

```python
# core/tests/factories.py
import factory
from django.contrib.auth import get_user_model
from core.models import Article, Category

User = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda o: f"{o.username}@test.com")
    password = factory.PostGenerationMethodCall("set_password", "pass123")


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"Category {n}")
    slug = factory.LazyAttribute(lambda o: o.name.lower().replace(" ", "-"))


class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f"Article {n}")
    slug = factory.LazyAttribute(lambda o: o.title.lower().replace(" ", "-"))
    body = factory.Faker("paragraph")
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
    status = "published"
```

```python
# core/tests/conftest.py
import pytest
from core.tests.factories import UserFactory, ArticleFactory


@pytest.fixture
def user(db):
    return UserFactory()


@pytest.fixture
def article(db, user):
    return ArticleFactory(author=user)
```

```python
# core/tests/test_models.py (pytest style)
import pytest
from core.models import Article


@pytest.mark.django_db
class TestArticleModel:
    def test_str(self, article):
        assert str(article) == article.title

    def test_published_manager(self, article):
        assert article in Article.published.all()
```

### Executar testes

```bash
# Django test runner
python manage.py test
python manage.py test core.tests.test_models      # módulo específico
python manage.py test core.tests.test_models.ArticleModelTest.test_str  # teste específico

# pytest
pytest
pytest core/tests/test_models.py -v
pytest -k "test_str"                               # por nome
pytest --cov=core --cov-report=html                # cobertura
```

---

## 20. Tarefas assíncronas com Celery

### Instalação

```bash
pip install celery redis
```

### Configuração

```python
# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# myproject/__init__.py
from .celery import app as celery_app

__all__ = ["celery_app"]
```

```python
# settings.py
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/0"
CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TIMEZONE = "America/Sao_Paulo"

# Beat (agendamento)
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "clean-old-drafts-daily": {
        "task": "core.tasks.clean_old_drafts",
        "schedule": crontab(hour=3, minute=0),     # 03:00 todo dia
    },
}
```

### Definir tasks

```python
# core/tasks.py
from celery import shared_task
from django.core.mail import send_mail
from django.utils import timezone


@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_notification_email(self, user_email, subject, message):
    try:
        send_mail(subject, message, "noreply@meusite.com", [user_email])
    except Exception as exc:
        self.retry(exc=exc)


@shared_task
def clean_old_drafts():
    from core.models import Article
    cutoff = timezone.now() - timezone.timedelta(days=90)
    count, _ = Article.objects.filter(status="draft", created_at__lt=cutoff).delete()
    return f"{count} rascunhos antigos removidos"
```

### Chamar tasks

```python
# Chamada assíncrona
send_notification_email.delay("user@email.com", "Assunto", "Mensagem")

# Com opções
send_notification_email.apply_async(
    args=["user@email.com", "Assunto", "Mensagem"],
    countdown=60,        # atraso de 60 segundos
)
```

### Executar workers

```bash
# Worker
celery -A myproject worker -l info

# Beat (agendador)
celery -A myproject beat -l info

# Ambos juntos (desenvolvimento)
celery -A myproject worker -B -l info
```

---

## 21. Django Channels (WebSockets)

### Instalação

```bash
pip install channels channels-redis
```

```python
# settings.py
INSTALLED_APPS = [
    "daphne",         # antes de django.contrib.staticfiles
    # ...
    "channels",
]

ASGI_APPLICATION = "myproject.asgi.application"

CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

### Consumer (WebSocket handler)

```python
# chat/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        message = data["message"]

        await self.channel_layer.group_send(
            self.room_group_name,
            {"type": "chat.message", "message": message},
        )

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({"message": event["message"]}))
```

### Routing

```python
# myproject/asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

from chat.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)
    ),
})
```

```python
# chat/routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", consumers.ChatConsumer.as_asgi()),
]
```

---

## 22. Internacionalização (i18n) e localização

```python
# settings.py
USE_I18N = True
USE_L10N = True
LANGUAGE_CODE = "pt-br"
TIME_ZONE = "America/Sao_Paulo"

from django.utils.translation import gettext_lazy as _

LANGUAGES = [
    ("pt-br", _("Português")),
    ("en", _("English")),
    ("es", _("Español")),
]

LOCALE_PATHS = [BASE_DIR / "locale"]

MIDDLEWARE = [
    # ...
    "django.middleware.locale.LocaleMiddleware",   # após SessionMiddleware, antes de CommonMiddleware
    # ...
]
```

### Marcar strings para tradução

```python
from django.utils.translation import gettext_lazy as _

class Article(models.Model):
    title = models.CharField(_("título"), max_length=200)

    class Meta:
        verbose_name = _("artigo")
        verbose_name_plural = _("artigos")
```

```html
{% load i18n %}
<h1>{% trans "Bem-vindo ao meu site" %}</h1>
<p>{% blocktrans with name=user.username %}Olá, {{ name }}!{% endblocktrans %}</p>
```

### Gerar e compilar traduções

```bash
django-admin makemessages -l en       # gera locale/en/LC_MESSAGES/django.po
django-admin compilemessages           # compila .po → .mo
```

---

## 23. Segurança

### Checklist de segurança

```bash
python manage.py check --deploy     # verifica configurações de segurança
```

### Configurações recomendadas

```python
# settings.py (produção)

# HTTPS
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")

# HSTS
SECURE_HSTS_SECONDS = 31536000            # 1 ano
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Cookies
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_AGE = 1209600              # 2 semanas
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True

# Proteções HTTP
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = "DENY"

# CORS (com django-cors-headers)
CORS_ALLOWED_ORIGINS = [
    "https://meudominio.com",
    "https://app.meudominio.com",
]

# Content Security Policy (com django-csp)
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "https://cdn.jsdelivr.net")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
```

### Proteção CSRF

```html
<!-- Em forms POST -->
<form method="post">
    {% csrf_token %}
    <!-- campos -->
</form>
```

```python
# Em chamadas AJAX (com cookie)
# O Django seta o cookie csrftoken; passe-o no header X-CSRFToken

# Isentar view específica do CSRF (use com cautela)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook(request):
    pass
```

### Rate limiting

```bash
pip install django-ratelimit
```

```python
from django_ratelimit.decorators import ratelimit

@ratelimit(key="ip", rate="10/m", block=True)
def login_view(request):
    pass
```

---

## 24. Performance e otimização

### Checklist de performance

1. **Evitar N+1 queries** — use `select_related` e `prefetch_related`.
2. **Indexar campos** usados em `filter`, `order_by` e lookups de FK.
3. **Usar `values()` / `values_list()`** quando não precisa do objeto completo.
4. **Paginação** — nunca retorne todos os registros de uma vez.
5. **Caching** — cache de views, querysets e template fragments.
6. **Database connection pooling** — use `django-db-connection-pool` ou PgBouncer.
7. **Compressão de assets** — WhiteNoise, django-compressor, ou CDN.
8. **Lazy evaluation** — querysets são lazy; evite forçar avaliação desnecessária.

### Django Debug Toolbar

```bash
pip install django-debug-toolbar
```

```python
# settings.py (development)
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
INTERNAL_IPS = ["127.0.0.1"]
```

```python
# urls.py
if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [path("__debug__/", include(debug_toolbar.urls))] + urlpatterns
```

### Database indexes

```python
class Article(models.Model):
    # ...
    class Meta:
        indexes = [
            models.Index(fields=["status", "-created_at"]),    # índice composto
            models.Index(fields=["slug"]),
            models.Index(
                fields=["status"],
                condition=models.Q(status="published"),
                name="idx_published_articles",                 # índice parcial (Postgres)
            ),
        ]
```

### Raw SQL e annotações complexas

```python
from django.db.models import Subquery, OuterRef, Exists
from django.db.models.functions import Coalesce, Lower

# Subquery
latest_article = Article.objects.filter(
    author=OuterRef("pk")
).order_by("-created_at")

users_with_latest = User.objects.annotate(
    latest_article_title=Subquery(latest_article.values("title")[:1])
)

# Raw SQL (quando necessário)
articles = Article.objects.raw(
    "SELECT * FROM core_article WHERE status = %s ORDER BY created_at DESC",
    ["published"],
)

# Cursor direto (para queries complexas)
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT COUNT(*) FROM core_article WHERE status = %s", ["published"])
    row = cursor.fetchone()
```

---

## 25. Async views (Django 4.1+)

```python
import asyncio
import httpx
from django.http import JsonResponse


async def fetch_external_data(request):
    async with httpx.AsyncClient() as client:
        resp1 = client.get("https://api.example.com/data1")
        resp2 = client.get("https://api.example.com/data2")
        r1, r2 = await asyncio.gather(resp1, resp2)

    return JsonResponse({
        "data1": r1.json(),
        "data2": r2.json(),
    })
```

> **Nota:** async views exigem um servidor ASGI (Daphne, Uvicorn, Hypercorn). O ORM do Django ainda é majoritariamente síncrono — use `sync_to_async` para wraps:

```python
from asgiref.sync import sync_to_async

@sync_to_async
def get_articles():
    return list(Article.objects.all()[:10])

async def async_view(request):
    articles = await get_articles()
    # ...
```

---

## 26. Deployment completo

### Dependências de produção

```txt
# requirements.txt
django>=5.0,<5.1
gunicorn
psycopg[binary]
django-environ
whitenoise
django-cors-headers
sentry-sdk
celery
redis
```

### Gunicorn

```bash
gunicorn myproject.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --threads 2 \
    --timeout 120 \
    --access-logfile - \
    --error-logfile -
```

> **Regra prática para workers:** `(2 × núcleos de CPU) + 1`.

### Uvicorn (ASGI)

```bash
uvicorn myproject.asgi:application \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4
```

### Nginx (proxy reverso)

```nginx
upstream django {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name meudominio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name meudominio.com;

    ssl_certificate     /etc/letsencrypt/live/meudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/meudominio.com/privkey.pem;

    client_max_body_size 10M;

    location /static/ {
        alias /var/www/myproject/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /var/www/myproject/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120;
    }
}
```

### systemd (gerenciar Gunicorn)

```ini
# /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn daemon para myproject
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myproject
ExecStart=/var/www/myproject/.venv/bin/gunicorn myproject.wsgi:application \
    --bind unix:/run/gunicorn.sock \
    --workers 4 \
    --timeout 120
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn
```

### Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  celery:
    build: .
    command: celery -A myproject worker -l info
    env_file: .env
    depends_on:
      - redis
      - db

  celery-beat:
    build: .
    command: celery -A myproject beat -l info
    env_file: .env
    depends_on:
      - redis

volumes:
  pgdata:
```

---

## 27. Boas práticas — resumo

| Área | Recomendação |
|---|---|
| **Projeto** | Um app por domínio de negócio; nomes descritivos |
| **Settings** | Separar por ambiente; variáveis de ambiente para segredos |
| **Models** | `__str__`, `Meta.ordering`, `get_absolute_url`, indexes |
| **QuerySets** | `select_related` / `prefetch_related`; evitar N+1 |
| **Views** | CBVs para CRUD, FBVs para lógica simples/custom |
| **Forms** | Validação no form (não na view); `clean_*` methods |
| **Templates** | Herança; `{% include %}` para parciais; não colocar lógica |
| **API** | DRF com serializers; paginação; throttling; versionamento |
| **Testes** | 80%+ de cobertura; factories; testar models, views, API |
| **Segurança** | `check --deploy`; HTTPS; CSRF; cookies seguros; CORS |
| **Cache** | Redis em produção; cache de views e fragmentos |
| **Performance** | Debug Toolbar; índices; paginação; lazy querysets |
| **Deploy** | Gunicorn + Nginx; systemd; Docker; CI/CD |
| **Logging** | Logging estruturado; Sentry para erros em produção |
| **Async** | Celery para tarefas pesadas; Channels para WebSockets |

---

## Comandos úteis — referência rápida

```bash
# Projeto
django-admin startproject myproject .
python manage.py startapp appname

# Banco de dados
python manage.py makemigrations
python manage.py migrate
python manage.py showmigrations
python manage.py sqlmigrate appname 0001
python manage.py dbshell

# Usuários
python manage.py createsuperuser
python manage.py changepassword username

# Estáticos
python manage.py collectstatic --noinput

# Shell interativo
python manage.py shell               # shell Python com Django carregado
python manage.py shell_plus          # (django-extensions) auto-import dos models

# Testes
python manage.py test
python manage.py test --parallel     # testes em paralelo
python manage.py test --verbosity=2

# Verificações
python manage.py check
python manage.py check --deploy      # checklist de segurança para produção

# Internacionalização
django-admin makemessages -l en
django-admin compilemessages

# Limpar sessões expiradas
python manage.py clearsessions
```

---

## Pacotes recomendados do ecossistema

| Pacote | Finalidade |
|---|---|
| `djangorestframework` | APIs REST |
| `django-filter` | Filtros para querysets e DRF |
| `django-cors-headers` | Configuração de CORS |
| `django-debug-toolbar` | Profiling em desenvolvimento |
| `django-extensions` | `shell_plus`, `show_urls`, etc. |
| `django-environ` | Variáveis de ambiente |
| `django-allauth` | Autenticação social (Google, GitHub, etc.) |
| `django-celery-beat` | Agendamento de tasks no banco |
| `django-storages` | Upload para S3, GCS, Azure Blob |
| `django-redis` | Backend Redis para cache/sessions |
| `django-silk` | Profiling de requests e queries |
| `django-health-check` | Health checks para monitoramento |
| `django-import-export` | Import/export CSV, Excel no Admin |
| `django-simple-history` | Histórico de alterações em models |
| `django-guardian` | Permissões por objeto (row-level) |
| `django-csp` | Content Security Policy headers |
| `sentry-sdk` | Monitoramento de erros |
| `factory-boy` | Factories para testes |
| `whitenoise` | Servir estáticos sem Nginx |

---

Referências: [documentação oficial do Django](https://docs.djangoproject.com/) · [Django REST Framework](https://www.django-rest-framework.org/) · [Celery](https://docs.celeryq.dev/) · [Django Channels](https://channels.readthedocs.io/) · [Classy CBV](https://ccbv.co.uk/)
