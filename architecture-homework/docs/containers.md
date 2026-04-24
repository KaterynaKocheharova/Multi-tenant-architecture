<!-- DOCS_NAV_START -->
[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Security](security.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)
<!-- DOCS_NAV_END -->

## Навігація в документі

- [Вебзастосунок (React)](#вебзастосунок-react)
  - [Відповідальність](#відповідальність)
- [Backend API (Node.js)](#backend-api-nodejs)
  - [Відповідальність](#відповідальність)
  - [Основні внутрішні модулі](#основні-внутрішні-модулі)
  - [Архітектурний стиль](#архітектурний-стиль)
- [База даних (PostgreSQL)](#база-даних-postgresql)
  - [Відповідальність](#відповідальність)
- [Комунікація контейнерів](#комунікація-контейнерів)
- [Ключовий принцип дизайну](#ключовий-принцип-дизайну)

<!-- DOCS_TOC_START -->
<!-- DOCS_TOC_END -->


# C4 Container

Система складається з трьох основних контейнерів:

## Вебзастосунок (React)

### Відповідальність

- Відображення таблиць учнів, викладачів, тенантів і подій із фільтрацією
- Форми для створення подій, реєстрацій і створення планів уроків
- UI для генерації попередньо заповнених звітів із можливістю редагування
- UI профілів учня та викладача
- Рендеринг інтерфейсу на основі ролей (RBAC)
- Комунікація з backend API

## Backend API (Node.js)

### Відповідальність

- Основна бізнес-логіка для всіх bounded contexts
- Надання REST API для frontend

### Основні внутрішні модулі

- AuthModule
- UsersModule
- EventsModule
- TeachersModule / StudentsModule
- LessonPlansModule
- ReportsModule
- TenantManagementModule

### Архітектурний стиль

- Модульний моноліт (під питанням, треба більше дослідження)
- CQRS-lite розділення (Command / Query сервіси)
- Патерн Strategy для варіацій типів подій/звітів

## База даних (PostgreSQL)

### Відповідальність

- Зберігання всіх даних застосунку
- Забезпечення реляційної цілісності між:
  - школами (тенантами)
  - користувачами
  - подіями
  - планами уроків тощо
- Підтримка складних запитів для звітності та агрегацій

## Комунікація контейнерів

- Web App → Backend API: REST/JSON через HTTP
- Backend API → Database: SQL-запити (ORM)
- Усередині Backend використовується модульна взаємодія між контекстами

## Ключовий принцип дизайну

- Доменна логіка живе у Backend API
- Web App є лише шаром представлення
- Database є шаром персистентності
