### 1. Инициализация Airflow

Перед началом работы убедитесь, что у вас настроены переменные окружения. Вы можете использовать файл .env или задать переменные другим способом.

#### Список переменных окружения:
# Учетные данные PostgreSQL
- `PG_USER`: имя пользователя для подключения к базе данных PostgreSQL (замените на ваше значение).
- `PG_PASSWORD`: пароль для пользователя PostgreSQL (замените на ваш пароль).
- `PG_DB`: название базы данных, которую вы будете использовать (например, ecommerce).
- `PG_HOST`: адрес сервера PostgreSQL (обычно это `localhost` или имя контейнера, если вы используете Docker).
- `PG_PORT`: порт, на котором работает PostgreSQL (по умолчанию 5432).

# Учетные данные MySQL
- `MYSQL_USER`: имя пользователя для подключения к базе данных MySQL (замените на ваше значение).
- `MYSQL_PASSWORD`: пароль для пользователя MySQL (замените на ваш пароль).
- `MYSQL_DB`: название базы данных для Airflow (например, airflow).
- `MYSQL_HOST`: адрес сервера MySQL (обычно это `localhost` или имя контейнера).
- `MYSQL_PORT`: порт, на котором работает MySQL (по умолчанию 3306).

#### Шаги для инициализации Airflow:
1. Убедитесь, что все переменные окружения настроены правильно.
2. Выполните следующую команду для инициализации Airflow:
   ```bash
   docker-compose -f docker-compose.init.yaml -d up
   ```

### 2. Запуск сервисов Airflow, Postgres, MySQL
Для поднятия необходимых сервисов выполните следующую команду:
```bash
docker-compose up
```
Это позволит вам запустить все сервисы, необходимые для работы проекта.

---

## Airflow Dags

### Вход в Airflow
После того как все сервисы запущены, перейдите по адресу: [http://localhost:8080/](http://localhost:8080/).  
Для входа используйте следующие учетные данные:
- Логин: `admin`
- Пароль: `admin`

### Описание доступных DAGs

#### 1. `initial_migration`
- Выполняется один раз для первоначальной настройки базы данных.
- Функциональность:
  - Создание таблиц в Postgres и MySQL.
  - Загрузка начальных данных в Postgres для дальнейшей работы.

#### 2. `transfer_postgres_mysql`
- Периодичность: ежедневно.
- Функциональность:
  - Перенос данных из Postgres в MySQL для обеспечения синхронизации данных.
- Настройка таблиц и полей для переноса:
  - Указана в файле ./dags/usecase/replicate_tables.yaml.

#### 3. `data_marts`
- Функциональность:
  - Создание аналитических витрин для анализа данных.

---

# Аналитическая витрина sales_analysis

## Описание
Витрина `sales_analysis` предназначена для интеграции данных о пользователях, заказах, продуктах и категориях в единую таблицу, которая будет использоваться для аналитических целей. Она предоставляет полное представление о взаимодействии клиентов с продуктами и заказами, что позволяет принимать обоснованные управленческие решения.

### Основные цели:
- Оценка доходов по продуктам, категориям и клиентам для выявления успешных направлений бизнеса.
- Выявление самых прибыльных клиентов и наиболее продаваемых продуктов для оптимизации ассортимента.
- Анализ статусов заказов для улучшения управления процессами выполнения заказов.
- Создание аналитических отчетов на основе интеграции данных из операционных таблиц, что способствует более глубокому пониманию бизнеса.

## Структура витрины

| Поле                   | Описание                                                   |
|-------------------------|-----------------------------------------------------------|
| user_id              | Уникальный идентификатор пользователя.                     |
| full_name            | Полное имя клиента, составленное из имени и фамилии.     |
| email                | Электронная почта клиента.                                 |
| phone                | Номер телефона клиента.                                   |
| loyalty_status       | Статус лояльности клиента (например, Gold, Silver, Bronze). |
| order_id             | Уникальный идентификатор заказа.                           |
| order_date           | Дата и время размещения заказа.                            |
| order_total          | Общая сумма заказа.                                        |
| product_id           | Уникальный идентификатор продукта.                         |
| product_name         | Название продукта.                                         |
| category_name        | Название категории, к которой принадлежит продукт.       |
| quantity             | Количество купленного продукта в данном заказе.          |
| price_per_unit       | Цена за единицу продукта.                                  |
| total_price_per_product | Общая стоимость продукта в рамках одного заказа.        |
| order_status         | Статус заказа (например, Pending, Completed, Canceled).  |

## SQL Исходник

```sql
-- Создание аналитической витрины "sales_analysis"
CREATE TABLE sales_analysis AS
SELECT
    u.user_id,
    CONCAT(u.first_name, ' ', u.last_name) AS full_name,
    u.email,
    u.phone,
    u.loyalty_status,
    o.order_id,
    o.order_date,
    o.total_amount AS order_total,
    p.product_id,
    p.name AS product_name,
    pc.name AS category_name,
    od.quantity,
    od.price_per_unit,
    od.quantity * od.price_per_unit AS total_price_per_product,
    o.status AS order_status
FROM
    orders o
JOIN
    users u ON o.user_id = u.user_id
JOIN
    orderDetails od ON o.order_id = od.order_id
JOIN
    products p ON od.product_id = p.product_id
JOIN
    productCategories pc ON p.category_id = pc.category_id;

## Примеры использования

### 1. Доходы по категориям продуктов
```sql
SELECT
    category_name,
    SUM(total_price_per_product) AS total_revenue
FROM
    sales_analysis
GROUP BY
    category_name
ORDER BY
    total_revenue DESC;
```

### 2. Топ-5 клиентов по доходам
```sql
SELECT
    full_name,
    email,
    SUM(order_total) AS total_spent
FROM
    sales_analysis
GROUP BY
    user_id, full_name, email
ORDER BY
    total_spent DESC
LIMIT 5;
```

### 3. Самые продаваемые продукты
```sql
SELECT
    product_name,
    category_name,
    SUM(quantity) AS total_sold
FROM
    sales_analysis
GROUP BY
    product_id, product_name, category_name
ORDER BY
    total_sold DESC
LIMIT 10;
```

### 4. Процент завершенных заказов
```sql
SELECT
    order_status,
    COUNT(*) AS order_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS percentage
FROM
    sales_analysis
GROUP BY
    order_status;
```
