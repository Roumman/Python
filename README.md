# Итоговый проект по Python

## Запуск

### 1. Инициализация Airflow

Настройте переменные окружения с помощью файла .env или другим способом.

#### Список переменных окружения:
# Учетные данные PostgreSQL
PGUSER=login
PGPASSWORD=pass
PGDB=ecommerce
PGHOST=postgresdb
PGPORT=5432

# Учетные данные MySQL
MYSQLUSER=airflow
MYSQLPASSWORD=airflow
MYSQLDB=airflow
MYSQLHOST=mysql
MYSQLPORT=3306

#### Шаги для инициализации Airflow:
1. Убедитесь, что переменные окружения настроены.
2. Выполните следующую команду для инициализации Airflow:
      docker-compose -f docker-compose.init.yaml -d up
   

### 2. Запуск сервисов Airflow, Postgres, MySQL
Чтобы поднять необходимые сервисы, выполните:
docker-compose up

---

## Airflow Dags

### Вход в Airflow
Перейдите по адресу: [http://localhost:8080/](http://localhost:8080/).  
Используйте следующие учетные данные для входа:
- Логин: admin
- Пароль: admin

### Описание доступных DAGs

#### 1. `initialmigration
- Выполняется один раз для первоначальной настройки.
- Функциональность:
  - Создание таблиц в Postgres и MySQL.
  - Загрузка начальных данных в Postgres.

#### 2. transferpostgresmysql
- Периодичность: ежедневно.
- Функциональность:
  - Перенос данных из Postgres в MySQL.
- Настройка таблиц и полей для переноса:
  - Указана в файле ./dags/usecase/replicate_tables.yaml.

#### 3. datamarts`
- Функциональность:
  - Создание аналитических витрин.

---

# Аналитическая витрина salesanalysis

## Описание
Витрина salesanalysis разработана для интеграции данных о пользователях, заказах, продуктах и категориях в единую таблицу для аналитики. Она дает полное представление, необходимое для создания отчетов, анализа продаж и принятия управленческих решений.

### Основные цели:
- Оценка доходов по продуктам, категориям и клиентам.
- Выявление наиболее прибыльных клиентов и самых продаваемых продуктов.
- Анализ статусов заказов для улучшения выполнения заказов.
- Создание аналитических отчетов на основе объединенных данных из операционных таблиц.

## Структура витрины

| Поле                   | Описание                                                   |
|-------------------------|-----------------------------------------------------------|
| userid              | Уникальный идентификатор пользователя.                     |
| fullname            | Полное имя клиента.                                        |
| email                | Email клиента.                                             |
| phone                | Телефон клиента.                                           |
| loyaltystatus       | Статус лояльности клиента (Gold, Silver, Bronze).     |
| orderid             | Уникальный идентификатор заказа.                           |
| orderdate           | Дата размещения заказа.                                    |
| ordertotal          | Общая сумма заказа.                                        |
| productid           | Уникальный идентификатор продукта.                         |
| productname         | Название продукта.                                         |
| categoryname        | Название категории продукта.                               |
| quantity             | Количество купленного продукта.                           |
| priceperunit       | Цена за единицу продукта.                                  |
| totalpriceperproduct | Общая стоимость продукта в рамках одного заказа.         |
| orderstatus         | Статус заказа (Pending, Completed, Canceled).        |

## SQL Исходник
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
SELECT
    category_name,
    SUM(total_price_per_product) AS total_revenue
FROM
    sales_analysis
GROUP BY
    category_name
ORDER BY
    total_revenue DESC;

### 2. Топ-5 клиентов по доходам
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

### 3. Самые продаваемые продукты
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

### 4. Процент завершенных заказов
SELECT
    order_status,
    COUNT(*) AS order_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS percentage
FROM
    sales_analysis
GROUP BY
    order_status;
