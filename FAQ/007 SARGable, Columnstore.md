## **Вопрос 14: Разница между INNER JOIN и LEFT JOIN в производительности**

Этот вопрос проверяет понимание физических операций, а не только логики.

### **Короткий ответ для собеседования**
> "INNER JOIN обычно быстрее, так как оптимизатор может менять порядок таблиц. LEFT JOIN всегда сохраняет порядок — левая таблица всегда первая. Разница в производительности зависит от данных: если большинство строк имеют соответствие — разница минимальна; если мало соответствий — LEFT JOIN может быть значительно медленнее из-за необходимости сохранять все строки левой таблицы."

### **Логическая vs Физическая разница**

```sql
-- Логически: INNER JOIN - только совпадающие строки
SELECT *
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
-- Только заказы с существующими клиентами

-- Логически: LEFT JOIN - все строки слева
SELECT *
FROM Orders o
LEFT JOIN Customers c ON o.CustomerID = c.CustomerID
-- Все заказы, даже если клиент удален (NULL в правых столбцах)
```

### **Как работают планы выполнения**

**INNER JOIN план (гибкий):**
```sql
-- Оптимизатор МОЖЕТ менять порядок
SELECT * 
FROM SmallTable s 
INNER JOIN BigTable b ON s.ID = b.ID

-- План может быть:
-- 1. SmallTable → Nested Loop → BigTable (если SmallTable маленькая)
-- 2. Hash Match (если оба большие)
-- 3. Merge Join (если отсортированы)
```

**LEFT JOIN план (фиксированный порядок):**
```sql
-- Порядок ВАЖЕН: левая таблица всегда ВНЕШНЯЯ
SELECT * 
FROM BigTable b 
LEFT JOIN SmallTable s ON b.ID = s.ID

-- План всегда:
-- BigTable (внешняя) → Nested Loop/Hash/Merge → SmallTable (внутренняя)
-- Нельзя поменять местами!
```

### **Сравнение производительности на практике**

```sql
-- Создаем тестовые данные
CREATE TABLE #Customers (
    CustomerID INT PRIMARY KEY,
    Name VARCHAR(50)
)

CREATE TABLE #Orders (
    OrderID INT IDENTITY PRIMARY KEY,
    CustomerID INT,
    Amount DECIMAL(10,2)
)

-- 1000 клиентов, 1M заказов
INSERT INTO #Customers 
SELECT TOP 1000 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)), 'Customer_' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS VARCHAR)
FROM sys.all_objects

INSERT INTO #Orders 
SELECT TOP 1000000 
    ABS(CHECKSUM(NEWID())) % 1000 + 1,  -- Случайный CustomerID 1-1000
    RAND() * 1000
FROM sys.all_objects a CROSS JOIN sys.all_objects b

-- Создаем индексы
CREATE INDEX IX_Orders_CustomerID ON #Orders(CustomerID)

-- Тест 1: Все строки имеют соответствие
SET STATISTICS IO, TIME ON

-- INNER JOIN
SELECT COUNT(*) 
FROM #Orders o
INNER JOIN #Customers c ON o.CustomerID = c.CustomerID
-- Reads: ~3000, Time: ~200ms

-- LEFT JOIN (те же строки)
SELECT COUNT(*) 
FROM #Orders o
LEFT JOIN #Customers c ON o.CustomerID = c.CustomerID
WHERE c.CustomerID IS NOT NULL  -- Эмулирует INNER
-- Reads: ~3000, Time: ~210ms (почти одинаково)
```

### **Когда LEFT JOIN значительно медленнее**

```sql
-- Тест 2: Мало соответствий (заказы без клиентов)
-- Добавляем заказы с несуществующими клиентами
INSERT INTO #Orders 
SELECT TOP 10000 
    9999 + ROW_NUMBER() OVER (ORDER BY (SELECT NULL)), -- CustomerID > 1000
    RAND() * 1000
FROM sys.all_objects

-- INNER JOIN (только существующие клиенты)
SELECT COUNT(*) 
FROM #Orders o
INNER JOIN #Customers c ON o.CustomerID = c.CustomerID
-- Reads: ~3000, Time: ~200ms

-- LEFT JOIN (все заказы)
SELECT COUNT(*) 
FROM #Orders o
LEFT JOIN #Customers c ON o.CustomerID = c.CustomerID
-- Должен сохранить ВСЕ строки #Orders
-- Reads: ~4000, Time: ~500ms (в 2.5 раза медленнее!)
```

### **Каверзные уточняющие вопросы**

1. **"Почему LEFT JOIN нельзя реорганизовать как INNER JOIN?"**
   - **Ответ**: Потому что LEFT JOIN должен сохранить все строки левой таблицы, даже без соответствий. Оптимизатор не может это гарантировать при смене порядка.

2. **"А если добавить WHERE по правой таблице?"**
   - **Ответ**: WHERE c.Column IS NOT NULL превращает LEFT JOIN в INNER JOIN на этапе оптимизации:
   ```sql
   -- Оптимизатор преобразует это:
   SELECT * FROM A LEFT JOIN B ON A.ID = B.ID WHERE B.Name = 'Test'
   -- В это:
   SELECT * FROM A INNER JOIN B ON A.ID = B.ID WHERE B.Name = 'Test'
   ```

3. **"Как ускорить LEFT JOIN?"**
   - **Ответ**: 
     - Индекс на join-колонке правой таблицы
     - Уменьшить размер левой таблицы (фильтрация до JOIN)
     - Использовать INNER JOIN если бизнес-логика позволяет

4. **"Что такое ANTI JOIN и как он связан с LEFT JOIN?"**
   - **Ответ**: Поиск строк без соответствия через LEFT JOIN:
   ```sql
   -- Найти клиентов без заказов
   SELECT c.* 
   FROM #Customers c
   LEFT JOIN #Orders o ON c.CustomerID = o.CustomerID
   WHERE o.OrderID IS NULL  -- ANTI JOIN
   -- План: Hash Match (Left Anti Semi Join)
   ```

### **Оптимизация LEFT JOIN**

```sql
-- Плохо: LEFT JOIN с последующей фильтрацией
SELECT o.*, c.Name
FROM #Orders o
LEFT JOIN #Customers c ON o.CustomerID = c.CustomerID
WHERE o.Amount > 100  -- Фильтрация ПОСЛЕ JOIN
-- Сначала JOIN 1M строк, потом фильтр

-- Лучше: фильтрация ДО JOIN
SELECT o.*, c.Name
FROM (
    SELECT * FROM #Orders WHERE Amount > 100  -- Сначала фильтруем
) o
LEFT JOIN #Customers c ON o.CustomerID = c.CustomerID
-- JOIN только отфильтрованных строк

-- Еще лучше: покрывающий индекс
CREATE INDEX IX_Orders_Amount_CustomerID 
ON #Orders(Amount) INCLUDE (CustomerID, OrderID, ...)
```

### **Особый случай: несколько LEFT JOIN**

```sql
-- Каскадные LEFT JOIN могут быть очень медленными
SELECT *
FROM A
LEFT JOIN B ON A.ID = B.A_ID      -- Все строки A
LEFT JOIN C ON B.ID = C.B_ID      -- Все строки B (уже включая NULL из первого JOIN!)
LEFT JOIN D ON C.ID = D.C_ID      -- И т.д.

-- Проблема: каждый LEFT JOIN увеличивает промежуточный набор
-- Решение: использовать INNER JOIN где возможно
SELECT *
FROM A
LEFT JOIN (
    B INNER JOIN C ON B.ID = C.B_ID  -- Если C всегда есть для B
) ON A.ID = B.A_ID
```

---

## **Вопрос 15: Что такое SARGable и как писать эффективные WHERE?**

### **Короткий ответ для собеседования**
> "SARGable (Search ARGument ABLE) — это предикат, который может использовать индекс для поиска. Не-SARGable условия заставляют сканировать всю таблицу. Основные правила: не применять функции к индексированным столбцам, избегать LIKE с начальным %, использовать эквивалентность вместо неравенств где возможно."

### **Что такое SARGable на примерах**

```sql
-- Создаем тестовую таблицу
CREATE TABLE Products (
    ProductID INT IDENTITY PRIMARY KEY,
    Name VARCHAR(100),
    Price DECIMAL(10,2),
    CreatedDate DATETIME,
    CategoryID INT
)

CREATE INDEX IX_Price ON Products(Price)
CREATE INDEX IX_CreatedDate ON Products(CreatedDate)
CREATE INDEX IX_Name ON Products(Name)

-- SARGable условия (могут использовать индекс)
SELECT * FROM Products WHERE Price = 100          -- Index Seek ✓
SELECT * FROM Products WHERE Price > 50 AND Price < 100 -- Range Seek ✓
SELECT * FROM Products WHERE CreatedDate >= '2024-01-01' -- Range Seek ✓
SELECT * FROM Products WHERE Name LIKE 'Apple%'  -- Index Seek ✓ (префиксный поиск)

-- Не-SARGable условия (сканирование)
SELECT * FROM Products WHERE Price * 1.1 > 110    -- Index Scan ✗ (функция слева)
SELECT * FROM Products WHERE YEAR(CreatedDate) = 2024 -- Index Scan ✗
SELECT * FROM Products WHERE Name LIKE '%Phone%'  -- Index Scan ✗ (начальный %)
SELECT * FROM Products WHERE SUBSTRING(Name, 1, 3) = 'App' -- Index Scan ✗
```

### **Преобразование не-SARGable в SARGable**

```sql
-- 1. Функции от столбцов
-- Плохо:
SELECT * FROM Products WHERE YEAR(CreatedDate) = 2024 AND MONTH(CreatedDate) = 1

-- Хорошо:
SELECT * FROM Products 
WHERE CreatedDate >= '2024-01-01' 
  AND CreatedDate < '2024-02-01'

-- 2. Математические операции
-- Плохо:
SELECT * FROM Products WHERE Price * 0.9 > 100

-- Хорошо:
SELECT * FROM Products WHERE Price > 100 / 0.9

-- 3. LIKE с начальным %
-- Плохо:
SELECT * FROM Products WHERE Name LIKE '%Phone%'

-- Хорошо (если нужно):
SELECT * FROM Products WHERE Name LIKE 'Phone%'  -- SARGable
-- Или полнотекстовый поиск:
CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT
CREATE FULLTEXT INDEX ON Products(Name) KEY INDEX PK_Products

SELECT * FROM Products 
WHERE CONTAINS(Name, '"Phone"')

-- 4. Функции строк
-- Плохо:
SELECT * FROM Products WHERE LEFT(Name, 5) = 'Apple'

-- Хорошо:
SELECT * FROM Products WHERE Name LIKE 'Apple%'

-- 5. Неявное преобразование типов
-- Плохо (если Price строковый):
CREATE TABLE Products2 (Price VARCHAR(20) INDEX IX_Price)
SELECT * FROM Products2 WHERE Price = 100  -- Преобразование INT→VARCHAR

-- Хорошо:
SELECT * FROM Products2 WHERE Price = '100'
```

### **Сложные случаи: OR и комбинации условий**

```sql
-- Проблема: OR может ломать SARGability
-- Плохо:
SELECT * FROM Products 
WHERE CategoryID = 1 OR Price < 50
-- Если есть индекс на CategoryID, но нет на Price (или наоборот)

-- Решение 1: UNION ALL (если индексы разные)
SELECT * FROM Products WHERE CategoryID = 1
UNION ALL
SELECT * FROM Products WHERE Price < 50 AND NOT (CategoryID = 1)

-- Решение 2: покрывающий индекс
CREATE INDEX IX_Category_Price ON Products(CategoryID, Price) INCLUDE (Name)

-- Решение 3: переписать логику
SELECT * FROM Products 
WHERE CategoryID = 1 OR (CategoryID <> 1 AND Price < 50)
```

### **Каверзные уточняющие вопросы**

1. **"А если нужен поиск по году и месяцу отдельно?"**
   - **Ответ**: Вычисляемый столбец с индексом:
   ```sql
   ALTER TABLE Products ADD YearMonth AS YEAR(CreatedDate) * 100 + MONTH(CreatedDate)
   CREATE INDEX IX_YearMonth ON Products(YearMonth)
   
   SELECT * FROM Products WHERE YearMonth = 202401
   ```

2. **"Почему WHERE Column IS NULL может быть не-SARGable?"**
   - **Ответ`: Зависит от индекса. Если индекс не включает NULL, то:
   ```sql
   CREATE INDEX IX_Price ON Products(Price) WHERE Price IS NOT NULL
   SELECT * FROM Products WHERE Price IS NULL  -- Index Scan (не использует индекс)
   ```

3. **"Что насчет IN vs OR?"**
   - **Ответ**: IN обычно преобразуется в OR. Для длинных списков лучше временная таблица:
   ```sql
   -- Плохо для длинного списка:
   SELECT * FROM Products WHERE CategoryID IN (1,2,3,4,5,6,7,8,9,10...)
   
   -- Лучше:
   CREATE TABLE #Categories (ID INT PRIMARY KEY)
   INSERT INTO #Categories VALUES (1),(2),(3)...
   
   SELECT p.* 
   FROM Products p
   INNER JOIN #Categories c ON p.CategoryID = c.ID
   ```

4. **"Как проверить, SARGable ли мой запрос?"**
   - **Ответ**: Смотреть план выполнения:
   ```sql
   SET SHOWPLAN_TEXT ON
   -- или
   SET STATISTICS PROFILE ON
   
   -- Index Seek = SARGable
   -- Index Scan = не-SARGable (или недостаточно селективно)
   ```

### **Оптимизация сложных предикатов**

```sql
-- Сложный пример: диапазон дат + дополнительные условия
-- Плохо:
SELECT * FROM Orders
WHERE (YEAR(OrderDate) = 2024 AND MONTH(OrderDate) = 1)
   OR (YEAR(OrderDate) = 2023 AND MONTH(OrderDate) = 12 AND Status = 'Completed')

-- Лучше: явные диапазоны
SELECT * FROM Orders
WHERE (OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01')
   OR (OrderDate >= '2023-12-01' 
       AND OrderDate < '2024-01-01' 
       AND Status = 'Completed')

-- Еще лучше: UNION ALL с разными индексами
SELECT * FROM Orders 
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01'

UNION ALL

SELECT * FROM Orders 
WHERE OrderDate >= '2023-12-01' 
  AND OrderDate < '2024-01-01'
  AND Status = 'Completed'
```

### **Специальные типы индексов для не-SARGable условий**

```sql
-- 1. Индексированное представление
CREATE VIEW vw_Products_Year
WITH SCHEMABINDING
AS
SELECT 
    YEAR(CreatedDate) AS SaleYear,
    COUNT_BIG(*) AS Count
FROM dbo.Products
GROUP BY YEAR(CreatedDate)

CREATE UNIQUE CLUSTERED INDEX IX_vw ON vw_Products_Year(SaleYear)

-- 2. Фильтрованный индекс
CREATE INDEX IX_HighPrice ON Products(Price)
WHERE Price > 1000  -- Только дорогие товары

-- 3. Columnstore индекс (для аналитических запросов)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Products ON Products
-- Хорошо для агрегаций, плохо для точечных запросов
```

### **Практическое правило SARGability**

> "Проверяйте план запроса. Если видите Index Scan вместо Seek — ищите:
> 1. Функции над индексированными столбцами (слева от оператора)
> 2. LIKE с начальным %
> 3. Неявное преобразование типов
> 4. OR с разными столбцами
> 5. Слишком сложные выражения"

**Пример быстрой проверки:**
```sql
-- Создаем копию с плохим запросом
SELECT * INTO #Test FROM Products
CREATE INDEX IX_Price ON #Test(Price)

-- Плохой запрос (Index Scan)
SELECT * FROM #Test WHERE Price + 10 > 100

-- Хороший запрос (Index Seek)  
SELECT * FROM #Test WHERE Price > 90
```

---

## **Вопрос 16: Как работают индексы Columnstore?**

### **Короткий ответ для собеседования**
> "Columnstore индексы хранят данные по столбцам, а не по строкам. Это позволяет эффективно сжимать данные (до 10x) и быстро выполнять аналитические запросы с агрегациями. Rowstore — для OLTP (точечные операции), Columnstore — для OLAP (аналитика). В SQL Server есть кластеризованные (CCI) и некластеризованные (NCCI) columnstore индексы."

### **Архитектура Columnstore vs Rowstore**

```sql
-- Rowstore (традиционный)
/*
Страница данных:
| ID | Name | Price | Date       | ... |
|----|------|-------|------------|-----|
| 1  | A    | 100   | 2024-01-01 | ... |
| 2  | B    | 200   | 2024-01-02 | ... |
| 3  | C    | 150   | 2024-01-03 | ... |
*/

-- Columnstore (по столбцам)
/*
Сегмент ID: | 1 | 2 | 3 | ... |
Сегмент Name: | A | B | C | ... |  
Сегмент Price: | 100 | 200 | 150 | ... |
Сегмент Date: | 2024-01-01 | 2024-01-02 | 2024-01-03 | ... |
*/
```

### **Создание и использование**

```sql
-- Создаем тестовую таблицу
CREATE TABLE Sales (
    SaleID INT IDENTITY,
    ProductID INT,
    SaleDate DATE,
    Quantity INT,
    Amount DECIMAL(10,2),
    RegionID INT,
    CustomerID INT
)

-- Вставляем много данных (миллионы строк)
INSERT INTO Sales 
SELECT 
    ABS(CHECKSUM(NEWID())) % 1000,
    DATEADD(DAY, -ABS(CHECKSUM(NEWID())) % 365, GETDATE()),
    ABS(CHECKSUM(NEWID())) % 100 + 1,
    RAND() * 1000,
    ABS(CHECKSUM(NEWID())) % 10 + 1,
    ABS(CHECKSUM(NEWID())) % 10000
FROM sys.all_objects a, sys.all_objects b, sys.all_objects c
WHERE ROWNUM <= 10000000

-- Создаем Clustered Columnstore Index (CCI)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales ON Sales
-- Вся таблица перестраивается в columnstore

-- Или Nonclustered Columnstore Index (NCCI) - SQL Server 2016+
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Sales 
ON Sales(ProductID, SaleDate, Quantity, Amount, RegionID)
-- Можно добавить к существующей rowstore таблице
```

### **Преимущества Columnstore**

```sql
-- 1. Сжатие (проверяем размер)
EXEC sp_spaceused 'Sales'
-- Rowstore: ~800 MB
-- Columnstore: ~80 MB (10x сжатие!)

-- 2. Batch Mode Execution (пакетный режим)
-- Аналитические запросы обрабатываются пакетами по ~900 строк
SELECT 
    ProductID,
    SUM(Quantity) AS TotalQty,
    AVG(Amount) AS AvgAmount
FROM Sales
WHERE SaleDate >= '2024-01-01'
GROUP BY ProductID
-- Batch Mode в плане: ✓

-- 3. Column Elimination (исключение столбцов)
SELECT ProductID, SUM(Quantity) 
FROM Sales
-- Читаются только сегменты ProductID и Quantity, не все столбцы
```

### **Ограничения Columnstore**

```sql
-- 1. Не для точечных запросов
SELECT * FROM Sales WHERE SaleID = 12345
-- Rowstore: Index Seek (быстро)
-- Columnstore: Index Scan (медленно)

-- 2. Ограничения DML
-- Обновления идут через Differential Store (дельту)
UPDATE Sales SET Amount = Amount * 1.1 WHERE RegionID = 1
-- Сначала в delta store, потом объединяется с основным

-- 3. Не все типы данных поддерживаются
-- Нельзя: MAX types, XML, geography, hierarchyid

-- 4. Совместимость с другими фичами
-- Нельзя: триггеры, изменение столбцов, уникальные ограничения
```

### **Каверзные уточняющие вопросы**

1. **"Когда использовать CCI vs NCCI?"**
   - **Ответ**:
     - **CCI** — когда таблица в основном для аналитики (SELECT), мало UPDATE/DELETE
     - **NCCI** — когда нужны и аналитика, и OLTP операции на одной таблице

2. **"Что такое Delta Store и Tuple Mover?"**
   - **Ответ**: Delta Store — rowstore для новых/измененных строк. Tuple Mover — фоновый процесс, который перемещает данные из Delta Store в основной columnstore.

3. **"Как мониторить фрагментацию columnstore?"**
   - **Ответ**:
   ```sql
   SELECT 
       OBJECT_NAME(object_id) AS TableName,
       row_group_id,
       total_rows,
       deleted_rows,
       state_desc,
       CASE 
           WHEN total_rows > 0 
           THEN (deleted_rows * 100.0 / total_rows) 
           ELSE 0 
       END AS fragmentation_percent
   FROM sys.dm_db_column_store_row_group_physical_stats
   WHERE object_id = OBJECT_ID('Sales')
   ORDER BY row_group_id
   
   -- REORGANIZE при фрагментации > 20%
   ALTER INDEX CCI_Sales ON Sales REORGANIZE
   ```

4. **"Можно ли создать columnstore на памяти-оптимизированной таблице?"**
   - **Ответ**: Да, с SQL Server 2017+:
   ```sql
   CREATE TABLE InMemorySales (
       SaleID INT NOT NULL PRIMARY KEY NONCLUSTERED,
       ProductID INT,
       Amount DECIMAL(10,2)
   ) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA)
   
   CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_InMemory 
   ON InMemorySales(ProductID, Amount)
   ```

### **Гибридные сценарии (HTAP)**

```sql
-- Operational Analytics: OLTP + аналитика на одной таблице
-- 1. Rowstore кластеризованный индекс для OLTP
CREATE CLUSTERED INDEX CI_Sales_OLTP ON Sales(SaleDate, SaleID)

-- 2. NCCI для аналитических запросов
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Sales_Analytics 
ON Sales(ProductID, SaleDate, Quantity, Amount)

-- OLTP операции быстрые (по кластеризованному индексу)
UPDATE Sales SET Quantity = 5 WHERE SaleID = 12345

-- Аналитические запросы используют NCCI
SELECT ProductID, SUM(Quantity) 
FROM Sales 
WHERE SaleDate >= '2024-01-01'
GROUP BY ProductID
```

### **Производительность: сравнение запросов**

```sql
-- Тестовый запрос: агрегация по месяцам
SET STATISTICS IO, TIME ON

-- 1. Rowstore с индексом
CREATE INDEX IX_Sales_Date ON Sales(SaleDate) INCLUDE (ProductID, Quantity)
SELECT 
    YEAR(SaleDate) AS SaleYear,
    MONTH(SaleDate) AS SaleMonth, 
    SUM(Quantity) AS TotalQty
FROM Sales
GROUP BY YEAR(SaleDate), MONTH(SaleDate)
ORDER BY SaleYear, SaleMonth
-- Время: ~5 сек, Reads: ~1M

-- 2. Columnstore (тот же запрос)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales ON Sales
SELECT 
    YEAR(SaleDate) AS SaleYear,
    MONTH(SaleDate) AS SaleMonth, 
    SUM(Quantity) AS TotalQty
FROM Sales
GROUP BY YEAR(SaleDate), MONTH(SaleDate)
ORDER BY SaleYear, SaleMonth
-- Время: ~0.5 сек, Reads: ~100K (в 10 раз быстрее!)
```

### **Практические рекомендации**

**Используйте Columnstore когда:**
- Таблица > 1M строк
- Частые аналитические запросы (GROUP BY, SUM, AVG)
- Много столбцов, но запросы используют подмножество
- Можно пожертвовать скоростью DML

**Не используйте Columnstore когда:**
- Частые точечные SELECT по ключу
- Много UPDATE/DELETE операций
- Требуются уникальные ограничения, триггеры
- Таблица маленькая (< 100K строк)

---
