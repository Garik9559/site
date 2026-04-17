# dekor-ks.ru — Рекомендуемый стек: Next.js + Django

## Архитектура

```
┌──────────────────────────────────────────────┐
│                   Cloudflare CDN              │
│              (кэш статики, SSL, WAF)         │
└──────────────┬───────────────┬───────────────┘
               │               │
    ┌──────────▼──────┐  ┌─────▼──────────┐
    │   Next.js 15    │  │   Django 5     │
    │   (Frontend)    │  │   (Backend)    │
    │   Vercel / VPS  │  │   VPS Docker   │
    │                 │  │                │
    │ - SSR/SSG/ISR   │  │ - REST API     │
    │ - React 19      │  │ - Admin panel  │
    │ - Tailwind CSS  │  │ - Wagtail CMS  │
    │ - Framer Motion │  │ - Auth/Perms   │
    └────────┬────────┘  └──────┬─────────┘
             │                  │
             │    API calls     │
             └────────┬─────────┘
                      │
          ┌───────────▼───────────┐
          │    PostgreSQL 17      │
          │    + Redis cache      │
          │    + Meilisearch      │
          └───────────────────────┘
                      │
          ┌───────────▼───────────┐
          │   S3 Storage          │
          │   (фото, DWG, STL,   │
          │    PDF, инструкции)   │
          └───────────────────────┘
```

## Django — модели данных

```python
# catalog/models.py

class Category(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    parent = models.ForeignKey('self', null=True, blank=True)
    icon = models.FileField()  # SVG иконка для mega-menu
    order = models.IntegerField(default=0)

class Material(models.Model):
    name = models.CharField()  # Архимрамор, Архикамень, Пенополистирол
    slug = models.SlugField()
    description = models.TextField()

class Product(models.Model):
    name = models.CharField()  # Карниз КР-001
    slug = models.SlugField()
    article = models.CharField()  # Арт. 123456
    category = models.ForeignKey(Category)
    description = models.TextField()

    # Размеры
    length = models.IntegerField(null=True)  # мм
    width = models.IntegerField(null=True)
    height = models.IntegerField(null=True)

    # Файлы
    dwg_file = models.FileField(null=True)
    pdf_spec = models.FileField(null=True)
    stl_model = models.FileField(null=True)
    mounting_instruction = models.FileField(null=True)

    # SEO
    meta_title = models.CharField()
    meta_description = models.TextField()

class ProductVariant(models.Model):
    """Вариация товара по материалу"""
    product = models.ForeignKey(Product, related_name='variants')
    material = models.ForeignKey(Material)
    price = models.DecimalField()  # цена за ед.
    price_unit = models.CharField()  # п.м. / шт.
    weight = models.DecimalField()  # кг
    stock = models.IntegerField()  # остаток
    is_available = models.BooleanField(default=True)

class ProductImage(models.Model):
    product = models.ForeignKey(Product, related_name='images')
    image = models.ImageField()
    is_main = models.BooleanField(default=False)
    order = models.IntegerField(default=0)

class ProductApplication(models.Model):
    """Фото товара в применении (на реальных домах)"""
    product = models.ForeignKey(Product, related_name='applications')
    project = models.ForeignKey('Project', null=True)
    image = models.ImageField()
    caption = models.CharField()

class RelatedProduct(models.Model):
    product = models.ForeignKey(Product, related_name='related_from')
    related = models.ForeignKey(Product, related_name='related_to')


# projects/models.py

class Project(models.Model):
    title = models.CharField()
    slug = models.SlugField()
    city = models.CharField()
    region = models.CharField()
    style = models.CharField()  # Классический, Итальянский...
    description = models.TextField()
    products_used = models.ManyToManyField(Product)

class ProjectImage(models.Model):
    project = models.ForeignKey(Project, related_name='images')
    image = models.ImageField()
    is_before = models.BooleanField(default=False)  # до/после


# orders/models.py

class Cart(models.Model):
    session_key = models.CharField()
    user = models.ForeignKey(User, null=True)
    created = models.DateTimeField(auto_now_add=True)

class CartItem(models.Model):
    cart = models.ForeignKey(Cart, related_name='items')
    variant = models.ForeignKey(ProductVariant)
    quantity = models.DecimalField()

class Order(models.Model):
    STATUS = [('new', 'Новый'), ('processing', 'В обработке'),
              ('confirmed', 'Подтверждён'), ('completed', 'Завершён')]
    cart = models.ForeignKey(Cart)
    status = models.CharField(choices=STATUS)
    name = models.CharField()
    phone = models.CharField()
    email = models.EmailField()
    city = models.CharField()
    comment = models.TextField(blank=True)
    total = models.DecimalField()
    # Интеграция с CRM
    crm_id = models.CharField(null=True)  # ID в Megaplan


# dealers/models.py

class Dealer(models.Model):
    company_name = models.CharField()
    city = models.CharField()
    region = models.CharField()
    address = models.TextField()
    phone = models.CharField()
    email = models.EmailField()
    website = models.URLField(blank=True)
    lat = models.FloatField()
    lng = models.FloatField()
    showroom_photo = models.ImageField(null=True)
    is_active = models.BooleanField(default=True)

class DealerApplication(models.Model):
    """Заявка на дилерство"""
    company_name = models.CharField()
    inn = models.CharField()
    contact_name = models.CharField()
    phone = models.CharField()
    email = models.EmailField()
    city = models.CharField()
    experience = models.TextField()
    status = models.CharField()  # new, reviewing, approved, rejected
```

## API endpoints (Django REST Framework)

```
GET  /api/v1/categories/                    — список категорий
GET  /api/v1/products/?category=&material=  — товары с фильтрами
GET  /api/v1/products/{slug}/               — карточка товара + варианты
GET  /api/v1/products/{slug}/related/       — связанные товары
GET  /api/v1/projects/                      — портфолио
GET  /api/v1/projects/{slug}/               — проект
GET  /api/v1/dealers/                       — дилеры
GET  /api/v1/blog/posts/                    — статьи
GET  /api/v1/blog/posts/{slug}/             — статья
POST /api/v1/blog/posts/{slug}/comments/    — комментарий
GET  /api/v1/materials/                     — материалы
POST /api/v1/cart/                          — создать корзину
POST /api/v1/cart/{id}/items/               — добавить в корзину
POST /api/v1/orders/                        — оформить заказ
POST /api/v1/calculator/                    — расчёт сметы
POST /api/v1/dealer-application/            — заявка на дилерство
GET  /api/v1/search/?q=                     — поиск (Meilisearch)
```

## Next.js — структура проекта

```
src/
├── app/
│   ├── layout.tsx              # Root layout (header + footer)
│   ├── page.tsx                # Главная
│   ├── catalog/
│   │   ├── page.tsx            # Каталог (все категории)
│   │   ├── [category]/
│   │   │   ├── page.tsx        # Листинг категории
│   │   │   └── [product]/
│   │   │       └── page.tsx    # Карточка товара
│   ├── services/
│   │   ├── page.tsx
│   │   └── [service]/page.tsx
│   ├── materials/
│   │   ├── page.tsx
│   │   └── [material]/page.tsx
│   ├── projects/
│   │   ├── page.tsx
│   │   └── [project]/page.tsx
│   ├── blog/
│   │   ├── page.tsx
│   │   └── [slug]/page.tsx
│   ├── dealers/page.tsx
│   ├── become-dealer/page.tsx
│   ├── about/page.tsx
│   ├── calculator/page.tsx
│   ├── cart/page.tsx
│   ├── contacts/page.tsx
│   └── faq/page.tsx
├── components/
│   ├── Header/
│   │   ├── Header.tsx
│   │   ├── MegaMenu.tsx
│   │   └── SearchBar.tsx
│   ├── ProductCard.tsx
│   ├── MaterialSwitcher.tsx
│   ├── DownloadButtons.tsx
│   ├── CartWidget.tsx
│   ├── DealerMap.tsx
│   └── Calculator/
│       └── CalculatorWizard.tsx
├── lib/
│   ├── api.ts                  # Django API client
│   ├── cart.ts                 # Cart store (Zustand)
│   └── search.ts              # Meilisearch client
└── styles/
    └── globals.css             # Tailwind
```

## SEO и AI-поиск

```
# /llms.txt — для AI-поисковиков
> Классический стиль — российский производитель архитектурного декора для фасада
> Каталог: 300+ элементов (карнизы, наличники, колонны, балюстрады)
> Материалы: архимрамор, архикамень, пенополистирол
> Услуги: 3D-проект, замер, производство, доставка, монтаж
> География: вся Россия, сеть дилеров в 72 регионах
> Контакт: 8 (800) 555-98-35, info@dekor-ks.ru

# Schema.org на каждой странице
- Product (с вариациями по материалу, ценами, наличием)
- Organization + LocalBusiness
- BreadcrumbList
- FAQPage (в блоге и FAQ)
- HowTo (инструкции монтажа)
- Article / BlogPosting
```

## Инфраструктура

```yaml
# docker-compose.yml (production)
services:
  django:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://...
      - REDIS_URL=redis://redis:6379
      - MEILISEARCH_URL=http://meilisearch:7700
    volumes:
      - media:/app/media

  postgres:
    image: postgres:17
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  meilisearch:
    image: getmeili/meilisearch:v1.10

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"

# Frontend деплоится на Vercel (или отдельный VPS)
```

## Этапы разработки

| Этап | Содержание | Срок |
|---|---|---|
| **1. MVP каталог** | Django модели + API + Next.js каталог + карточка товара + корзина | 6-8 недель |
| **2. Контент** | Блог (Wagtail), страницы услуг, материалов, о компании | 2-3 недели |
| **3. Бизнес-логика** | Калькулятор, заявки в CRM, email-уведомления | 2-3 недели |
| **4. Дилерская сеть** | Карта дилеров, кабинет дилера, заявка на дилерство | 2-3 недели |
| **5. SEO и запуск** | Schema.org, sitemap, llms.txt, миграция контента, редиректы | 1-2 недели |
| **6. Доработки** | 3D-превью, онлайн-оплата, расширенная аналитика | ongoing |
