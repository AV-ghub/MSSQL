## **Вопрос 12: Как работают оконные функции (ROW_NUMBER, RANK, LAG)?**

Этот вопрос проверяет знание современных возможностей T-SQL для аналитики.

### **Короткий ответ для собеседования**
> "Оконные функции выполняют вычисления над набором строк (окном), связанных с текущей строкой. Они не сворачивают результат как GROUP BY, а добавляют вычисляемые столбцы к каждой строке. ROW_NUMBER — уникальная нумерация, RANK — нумерация с пропусками при одинаковых значениях, LAG/LEAD — доступ к предыдущим/следующим строкам."

### **Базовый синтаксис и отличия от GROUP BY**

```sql
-- GROUP BY сворачивает строки
SELECT DepartmentID, AVG(Salary) AS AvgSalary
FROM Employees
GROUP BY DepartmentID  -- 1 строка на отдел

-- Оконные функции сохраняют все строки
SELECT 
    EmployeeID, Name, DepartmentID, Salary,
    AVG(Salary) OVER (PARTITION BY DepartmentID) AS AvgSalaryInDept
FROM Employees  -- Все строки остаются!
```

### **ROW_NUMBER, RANK, DENSE_RANK - ключевые отличия**

```sql
-- Тестовые данные
CREATE TABLE Sales (
    SaleID INT IDENTITY,
    SalesPerson VARCHAR(50),
    Amount DECIMAL(10,2),
    SaleDate DATE
)

INSERT INTO Sales VALUES 
('Иван', 1000, '2024-01-10'),
('Иван', 1000, '2024-01-12'),  -- Дубликат суммы
('Петр', 1500, '2024-01-11'),
('Мария', 800, '2024-01-10'),
('Иван', 1200, '2024-01-15'),
('Петр', 900, '2024-01-14'),
('Мария', 950, '2024-01-16')

-- Сравнение функций нумерации
SELECT 
    SalesPerson,
    Amount,
    ROW_NUMBER() OVER (ORDER BY Amount DESC) AS RowNum,      -- 1,2,3,4,5,6,7
    RANK() OVER (ORDER BY Amount DESC) AS RankNum,          -- 1,1,3,4,5,6,7 (пропуск 2)
    DENSE_RANK() OVER (ORDER BY Amount DESC) AS DenseRank  -- 1,1,2,3,4,5,6 (без пропусков)
FROM Sales
```

### **PARTITION BY - "окна" внутри окон**

```sql
-- Нумерация ВНУТРИ каждого продавца
SELECT 
    SalesPerson,
    SaleDate,
    Amount,
    ROW_NUMBER() OVER (
        PARTITION BY SalesPerson 
        ORDER BY Amount DESC
    ) AS RankInPerson,  -- Для каждого продавца своя нумерация с 1
    
    -- Сравнение с общим зачетом
    ROW_NUMBER() OVER (
        ORDER BY Amount DESC
    ) AS OverallRank
FROM Sales
/*
SalesPerson | Amount | RankInPerson | OverallRank
Иван        | 1200   | 1            | 1
Иван        | 1000   | 2            | 2  
Иван        | 1000   | 3            | 3
Петр        | 1500   | 1            | 1
Петр        | 900    | 2            | 6
*/
```

### **LAG и LEAD - доступ к соседним строкам**

```sql
-- Анализ динамики продаж по дням
SELECT 
    SalesPerson,
    SaleDate,
    Amount,
    -- Предыдущая продажа того же продавца
    LAG(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
    ) AS PreviousAmount,
    
    -- Изменение к предыдущей
    Amount - LAG(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
    ) AS ChangeFromPrevious,
    
    -- Процент изменения
    CASE 
        WHEN LAG(Amount) OVER (PARTITION BY SalesPerson ORDER BY SaleDate) > 0
        THEN (Amount - LAG(Amount) OVER (PARTITION BY SalesPerson ORDER BY SaleDate)) * 100.0 
             / LAG(Amount) OVER (PARTITION BY SalesPerson ORDER BY SaleDate)
        ELSE NULL
    END AS PercentChange,
    
    -- Следующая продажа
    LEAD(SaleDate, 1, '9999-12-31') OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
    ) AS NextSaleDate
FROM Sales
ORDER BY SalesPerson, SaleDate
```

### **Агрегатные функции как оконные**

```sql
-- Скользящее среднее и накопительный итог
SELECT 
    SalesPerson,
    SaleDate,
    Amount,
    -- Среднее за все предыдущие продажи этого продавца
    AVG(Amount) OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningAvg,
    
    -- Накопительная сумма
    SUM(Amount) OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
        ROWS UNBOUNDED PRECEDING
    ) AS RunningTotal,
    
    -- Скользящее среднее за последние 2 продажи
    AVG(Amount) OVER (
        PARTITION BY SalesPerson 
        ORDER BY SaleDate
        ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS Last2Avg
FROM Sales
```

### **Каверзные уточняющие вопросы**

1. **"В чем разница между ROWS и RANGE?"**
   - **Ответ**: ROWS — физические строки, RANGE — логические по значениям ORDER BY. RANGE медленнее и может давать неожиданные результаты при дубликатах.
   
   ```sql
   SELECT 
       Amount,
       SUM(Amount) OVER (ORDER BY Amount ROWS UNBOUNDED PRECEDING) AS RowsSum,
       SUM(Amount) OVER (ORDER BY Amount RANGE UNBOUNDED PRECEDING) AS RangeSum
   FROM Sales
   -- При Amount=1000 (2 строки): RowsSum считает построчно, RangeSum сложит обе 1000 сразу
   ```

2. **"Можно ли использовать оконные функции в WHERE?"**
   - **Ответ**: Нет, напрямую нельзя. Нужно использовать CTE или подзапрос:
   ```sql
   -- Не работает
   SELECT * FROM Sales 
   WHERE ROW_NUMBER() OVER (ORDER BY Amount) <= 3
   
   -- Работает
   WITH Numbered AS (
       SELECT *, ROW_NUMBER() OVER (ORDER BY Amount) AS rn
       FROM Sales
   )
   SELECT * FROM Numbered WHERE rn <= 3
   ```

3. **"Как получить топ-N записей для каждой группы?"**
   - **Ответ**: Классический шаблон с ROW_NUMBER:
   ```sql
   -- Топ-2 продажи каждого продавца
   WITH Ranked AS (
       SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY SalesPerson 
               ORDER BY Amount DESC
           ) AS rn
       FROM Sales
   )
   SELECT * FROM Ranked WHERE rn <= 2
   ```

4. **"Что будет при ORDER BY в OVER без PARTITION BY?"**
   - **Ответ**: Весь набор как одна партиция. Если ORDER BY опустить, окно = все строки таблицы.

### **Производительность и индексы**

```sql
-- Индексы для оконных функций
CREATE INDEX IX_Sales_PersonDate 
ON Sales(SalesPerson, SaleDate) 
INCLUDE (Amount)

-- Плохой запрос (сканирование + сортировка)
SELECT 
    SalesPerson,
    LAG(Amount) OVER (PARTITION BY SalesPerson ORDER BY SaleDate)
FROM Sales
-- Без индекса: Table Scan + Sort

-- Хороший запрос (seek по индексу)
SELECT 
    SalesPerson,
    LAG(Amount) OVER (PARTITION BY SalesPerson ORDER BY SaleDate)
FROM Sales
-- С индексом: Index Seek (уже отсортировано!)
```

### **Практические примеры**

**Пример 1: Поиск дубликатов**
```sql
-- Найти полные дубликаты
WITH Duplicates AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY Name, Email, Phone 
            ORDER BY (SELECT NULL)
        ) AS dup_num
    FROM Customers
)
SELECT * FROM Duplicates WHERE dup_num > 1

-- Альтернатива для удаления дублей
DELETE FROM Duplicates WHERE dup_num > 1
```

**Пример 2: Gap analysis (поиск пропусков)**
```sql
-- Найти пропуски в нумерации заказов
WITH OrdersWithPrev AS (
    SELECT 
        OrderID,
        LAG(OrderID) OVER (ORDER BY OrderID) AS PrevOrderID
    FROM Orders
)
SELECT 
    PrevOrderID + 1 AS GapStart,
    OrderID - 1 AS GapEnd
FROM OrdersWithPrev
WHERE OrderID - PrevOrderID > 1
```

**Пример 3: Расчет процента от общего**
```sql
-- Доля каждого отдела в общем бюджете
SELECT 
    DepartmentID,
    Budget,
    Budget * 100.0 / SUM(Budget) OVER () AS PercentOfTotal,
    -- Внутри региона
    Budget * 100.0 / SUM(Budget) OVER (PARTITION BY RegionID) AS PercentInRegion
FROM Departments
```

---

## **Вопрос 13: Проблемы с динамическим SQL и SQL injection**

### **Короткий ответ для собеседования**
> "Динамический SQL — построение строк запроса во время выполнения. Главная опасность — SQL injection, когда пользовательский ввод исполняется как код. Защита: параметризация (sp_executesql), проверка ввода, минимальные права, экранирование. Никогда не конкатенируйте пользовательский ввод напрямую!"

### **Как происходит SQL injection**

```sql
-- УЯЗВИМЫЙ КОД (НИКОГДА ТАК НЕ ДЕЛАЙТЕ!)
DECLARE @UserName NVARCHAR(100) = N'admin'' OR 1=1--'
DECLARE @SQL NVARCHAR(MAX)

-- Прямая конкатенация
SET @SQL = N'SELECT * FROM Users WHERE UserName = ''' + @UserName + ''''
-- Получается: SELECT * FROM Users WHERE UserName = 'admin' OR 1=1--'
-- Комментарий (--) отрезает все проверки пароля!

EXEC(@SQL)
```

### **Правильная параметризация**

```sql
-- БЕЗОПАСНЫЙ КОД
DECLARE @UserName NVARCHAR(100) = N'admin'' OR 1=1--'
DECLARE @SQL NVARCHAR(MAX)
DECLARE @Params NVARCHAR(MAX)

-- Параметризованный запрос
SET @SQL = N'SELECT * FROM Users WHERE UserName = @UserNameParam'
SET @Params = N'@UserNameParam NVARCHAR(100)'

-- Параметры передаются отдельно
EXEC sp_executesql @SQL, @Params, @UserNameParam = @UserName
-- Теперь @UserName рассматривается как ДАННЫЕ, а не код
```

### **QUOTENAME и безопасная динамика**

```sql
-- Динамические имена объектов (таблиц, столбцов)
DECLARE @TableName NVARCHAR(128) = N'Users'
DECLARE @ColumnName NVARCHAR(128) = N'UserName'
DECLARE @SQL NVARCHAR(MAX)

-- ОПАСНО
SET @SQL = N'SELECT * FROM ' + @TableName + ' WHERE ' + @ColumnName + ' = @Value'

-- БЕЗОПАСНО: QUOTENAME для имен объектов
SET @SQL = N'SELECT * FROM ' + QUOTENAME(@TableName) + 
           N' WHERE ' + QUOTENAME(@ColumnName) + N' = @Value'

DECLARE @Params NVARCHAR(MAX) = N'@Value NVARCHAR(100)'
EXEC sp_executesql @SQL, @Params, @Value = N'admin'
```

### **Типичные сценарии и защита**

**Сценарий 1: Динамическая сортировка (ORDER BY)**
```sql
-- Проблема: ORDER BY нельзя параметризовать напрямую
DECLARE @SortColumn NVARCHAR(100) = N'UserName; DROP TABLE Users--'
DECLARE @SortOrder NVARCHAR(4) = N'ASC'

-- Защита: белый список
IF @SortColumn NOT IN ('UserName', 'Email', 'CreatedDate')
    SET @SortColumn = 'UserName'
    
IF UPPER(@SortOrder) NOT IN ('ASC', 'DESC')
    SET @SortOrder = 'ASC'

SET @SQL = N'SELECT * FROM Users ORDER BY ' + 
           QUOTENAME(@SortColumn) + ' ' + @SortOrder
EXEC sp_executesql @SQL
```

**Сценарий 2: Динамический PIVOT**
```sql
-- Динамические имена столбцов для PIVOT
DECLARE @Columns NVARCHAR(MAX), @SQL NVARCHAR(MAX)

-- Собираем имена столбцов с QUOTENAME
SELECT @Columns = COALESCE(@Columns + ', ', '') + QUOTENAME(Year)
FROM (SELECT DISTINCT Year FROM Sales) AS Years

SET @SQL = N'
SELECT Product, ' + @Columns + '
FROM (
    SELECT Product, Year, Amount 
    FROM Sales
) AS Source
PIVOT (
    SUM(Amount) FOR Year IN (' + @Columns + ')
) AS PivotTable'

EXEC sp_executesql @SQL
```

### **Каверзные уточняющие вопросы**

1. **"Можно ли использовать EXEC() вместо sp_executesql?"**
   - **Ответ**: sp_executesql лучше — поддерживает параметризацию и кэширование планов. EXEC() только для статических строк.

2. **"Как защититься от второго порядка SQL injection?"**
   - **Ответ**: Даже данные из БД могут быть вредоносными:
   ```sql
   -- Злоумышленник сохраняет: '; DELETE FROM Users--
   INSERT INTO Config (SettingName, SettingValue) 
   VALUES ('SearchQuery', '''; DELETE FROM Users--')
   
   -- Позже уязвимый код:
   DECLARE @query NVARCHAR(100)
   SELECT @query = SettingValue FROM Config WHERE SettingName = 'SearchQuery'
   EXEC('SELECT * FROM Products WHERE Name LIKE ''' + @query + '''')
   -- Решение: параметризация всегда!
   ```

3. **"Что такое dynamic SQL blinding?"**
   - **Ответ**: Атакующий определяет структуру БД по времени ответа или ошибкам:
   ```sql
   -- Пример уязвимости
   IF EXISTS(SELECT * FROM sysobjects WHERE name = 'Users')
       WAITFOR DELAY '00:00:05'  -- Задержка выдает информацию!
   ```

4. **"Как минимизировать права для динамического SQL?"**
   - **Ответ**: Использовать модули с подписанными сертификатами или EXECUTE AS:
   ```sql
   CREATE PROCEDURE DynamicSearch
       @SearchTerm NVARCHAR(100)
   WITH EXECUTE AS 'LimitedUser'  -- Минимальные права
   AS
   BEGIN
       DECLARE @SQL NVARCHAR(MAX)
       SET @SQL = N'SELECT * FROM Products WHERE Name LIKE @Term'
       EXEC sp_executesql @SQL, N'@Term NVARCHAR(100)', @Term = @SearchTerm
   END
   ```

### **Техники защиты**

**1. Хранимые процедуры с динамическим SQL**
```sql
CREATE PROCEDURE SafeDynamicSearch
    @ColumnName NVARCHAR(128),
    @SearchValue NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Белый список столбцов
    IF @ColumnName NOT IN ('Name', 'Email', 'Phone')
        THROW 50000, 'Invalid column name', 1;
    
    DECLARE @SQL NVARCHAR(MAX)
    SET @SQL = N'SELECT * FROM Users WHERE ' + 
               QUOTENAME(@ColumnName) + N' = @Value'
    
    EXEC sp_executesql @SQL, N'@Value NVARCHAR(100)', @Value = @SearchValue
END
```

**2. Подписание процедур сертификатом**
```sql
-- 1. Создаем пользователя с минимальными правами
CREATE USER DynamicSQLUser WITHOUT LOGIN
GRANT SELECT ON Users TO DynamicSQLUser

-- 2. Создаем сертификат
CREATE CERTIFICATE DynamicSQLCert
   ENCRYPTION BY PASSWORD = 'StrongPassword'
   WITH SUBJECT = 'Certificate for dynamic SQL'

-- 3. Подписываем процедуру
ADD SIGNATURE TO SafeDynamicSearch
   BY CERTIFICATE DynamicSQLCert
   WITH PASSWORD = 'StrongPassword'
```

**3. Аудит и мониторинг**
```sql
-- Включить аудит
CREATE DATABASE AUDIT SPECIFICATION AuditDynamicSQL
FOR SERVER AUDIT SQLServerAudit
ADD (EXECUTE ON DATABASE::YourDB BY PUBLIC)
WITH (STATE = ON)

-- Мониторинг подозрительных запросов
SELECT 
    text, 
    execution_count,
    last_execution_time
FROM sys.dm_exec_query_stats
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE text LIKE '%EXEC(%'
   OR text LIKE '%sp_executesql%'
ORDER BY last_execution_time DESC
```

### **Практические тесты на безопасность**

```sql
-- Тест 1: Простая инъекция
EXEC SafeDynamicSearch 'Name', N'admin'' OR 1=1--'
-- Результат: ошибка или пустой результат

-- Тест 2: Инъекция в имя столбца  
EXEC SafeDynamicSearch 'Name; DELETE FROM Users--', N'test'
-- Результат: ошибка "Invalid column name"

-- Тест 3: UNION-based инъекция
EXEC SafeDynamicSearch 'Name', N'admin'' UNION SELECT * FROM Users--'
-- Результат: ищет буквальную строку со всем текстом

-- Тест 4: Time-based blind
EXEC SafeDynamicSearch 'Name', N'admin''; IF (SELECT COUNT(*) FROM Users) > 0 WAITFOR DELAY ''00:00:05''--'
-- Результат: параметризация предотвращает выполнение
```

### **Главные правила безопасности**
1. **Всегда используйте sp_executesql с параметрами**
2. **QUOTENAME для имен объектов** (таблиц, столбцов)
3. **Белые списки** для динамических частей
4. **Минимальные права** (EXECUTE AS, подписанные модули)
5. **Валидация ввода** на всех уровнях
6. **Логирование и аудит** подозрительной активности
7. **Регулярные тесты** на уязвимости

---
