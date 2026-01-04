## **Что такое Parameter Sniffing?**

> **Parameter Sniffing — это когда SQL Server "нюхает" (анализирует) значения параметров при ПЕРВОЙ компиляции запроса и создает план, оптимизированный под эти значения. Этот план затем кэшируется и используется для всех последующих вызовов.**

### **Как это выглядит на практике**

```sql
-- Допустим, есть таблица заказов с индексом по дате
CREATE TABLE Orders (
    OrderID INT IDENTITY PRIMARY KEY,
    CustomerID INT,
    OrderDate DATE,
    Amount DECIMAL(10,2)
)

CREATE INDEX IX_Orders_Date ON Orders(OrderDate)

-- Заполняем данными
-- 1,000,000 старых заказов (2020-2023)
-- 100 свежих заказов (2024)
```

### **Проблема в действии**

```sql
-- Процедура поиска заказов
CREATE PROCEDURE GetOrdersByDate
    @StartDate DATE
AS
SELECT * FROM Orders 
WHERE OrderDate >= @StartDate

-- ПЕРВЫЙ вызов (план кэшируется)
EXEC GetOrdersByDate @StartDate = '2024-01-01'
-- 100 строк → Index Seek (быстро)
-- План: NonClustered Index Seek + Key Lookup (стоимость: 0.05)

-- ВТОРОЙ вызов (использует КЭШИРОВАННЫЙ план)
EXEC GetOrdersByDate @StartDate = '2020-01-01'
-- 1,000,100 строк → Тоже Index Seek + Key Lookup!
-- Но должен быть Clustered Index Scan!
-- Результат: 100,000+ Key Lookups, запрос выполняется 10 секунд вместо 2
```

## **Почему это происходит?**

SQL Server делает так:
1. **Первый вызов**: компилирует план для `@StartDate = '2024-01-01'`
2. **Оценивает**: "вернет ~100 строк" → выбирает Index Seek + Key Lookup
3. **Кэширует** этот план
4. **Все последующие вызовы** используют тот же план, даже для других параметров

## **"Проблема параметризации" vs Parameter Sniffing**

Это две разные, но связанные проблемы:

### **1. Parameter Sniffing (для хранимых процедур)**
```sql
-- Только для хранимых процедур/параметризованных запросов
CREATE PROCEDURE Test @Param INT
AS
SELECT * FROM Table WHERE Column = @Param
-- План кэшируется с учетом ПЕРВОГО значения @Param
```

### **2. Auto-Parameterization (автопараметризация)**
```sql
-- SQL Server сам параметризует ad-hoc запросы
SELECT * FROM Orders WHERE OrderID = 12345
-- Может стать: SELECT * FROM Orders WHERE OrderID = @1

-- Проблема: разные константы → разный план
SELECT * FROM Orders WHERE OrderID = 1  -- План A
SELECT * FROM Orders WHERE OrderID = 999999  -- План B (если 999999 нет)
-- Хотя логически запросы одинаковые!
```

## **Реальные симптомы Parameter Sniffing**

```sql
-- Мониторинг проблемных планов
SELECT 
    ps.plan_handle,
    ps.execution_count,
    ps.total_worker_time / ps.execution_count AS avg_cpu_ms,
    ps.total_elapsed_time / ps.execution_count AS avg_duration_ms,
    ps.total_logical_reads / ps.execution_count AS avg_reads,
    SUBSTRING(st.text, 
        (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset 
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset 
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_procedure_stats ps
CROSS APPLY sys.dm_exec_sql_text(ps.sql_handle) st
WHERE ps.database_id = DB_ID()
    AND ps.object_id = OBJECT_ID('GetOrdersByDate')
    AND ps.execution_count > 100
    AND ps.total_worker_time / ps.execution_count > 100  -- > 100ms в среднем
ORDER BY (ps.max_worker_time - ps.min_worker_time) DESC  -- Максимальный разброс
```

## **Стратегии решения Parameter Sniffing**

### **Стратегия 1: OPTION (RECOMPILE)**
```sql
-- Перекомпиляция при каждом вызове
CREATE PROCEDURE GetOrdersByDate @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate >= @StartDate
OPTION (RECOMPILE)

-- Плюсы: всегда оптимальный план
-- Минусы: overhead компиляции (5-50ms на запрос)
-- Когда использовать: запросы с сильно меняющейся селективностью
```

### **Стратегия 2: OPTION (OPTIMIZE FOR UNKNOWN)**
```sql
-- Использовать "усредненную" статистику
CREATE PROCEDURE GetOrdersByDate @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate >= @StartDate
OPTION (OPTIMIZE FOR UNKNOWN)

-- План будет основан на средней селективности столбца
-- Например: "вероятно вернет 30% строк" → выберет Table Scan
```

### **Стратегия 3: OPTION (OPTIMIZE FOR (@value))**
```sql
-- Оптимизировать под типичный сценарий
CREATE PROCEDURE GetOrdersByDate @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate >= @StartDate
OPTION (OPTIMIZE FOR (@StartDate = '2024-01-01'))

-- Всегда использует план для свежих данных
-- Плохо, если часто запрашивают старые данные
```

### **Стратегия 4: Локальные переменные (старый трюк)**
```sql
-- Обход sniffing через локальную переменную
CREATE PROCEDURE GetOrdersByDate @StartDate DATE
AS
DECLARE @LocalDate DATE = @StartDate
SELECT * FROM Orders WHERE OrderDate >= @LocalDate

-- Оптимизатор не знает значение @LocalDate
-- Использует статистику плотности (average across all values)
-- Часто приводит к усредненному (неоптимальному) плану
```

### **Стратегия 5: Динамический SQL с RECOMPILE**
```sql
-- RECOMPILE только для сложных случаев
CREATE PROCEDURE GetOrdersByDate 
    @StartDate DATE,
    @UseRecompile BIT = 0
AS
DECLARE @SQL NVARCHAR(MAX)
SET @SQL = N'SELECT * FROM Orders WHERE OrderDate >= @DateParam'

IF @UseRecompile = 1
    SET @SQL = @SQL + N' OPTION (RECOMPILE)'

EXEC sp_executesql @SQL, N'@DateParam DATE', @DateParam = @StartDate
```

### **Стратегия 6: Разделение на несколько процедур**
```sql
-- Разные процедуры для разных сценариев
CREATE PROCEDURE GetRecentOrders  -- Для свежих данных
    @DaysBack INT = 7
AS
SELECT * FROM Orders 
WHERE OrderDate >= DATEADD(DAY, -@DaysBack, GETDATE())
-- Index Seek

CREATE PROCEDURE GetHistoricalOrders  -- Для старых данных
    @StartDate DATE,
    @EndDate DATE
AS
SELECT * FROM Orders 
WHERE OrderDate BETWEEN @StartDate AND @EndDate
-- Table Scan, возможно с Columnstore
```

## **Практические примеры решения**

### **Пример 1: Поиск по диапазону дат**
```sql
-- Исходная проблема
CREATE PROCEDURE SearchOrders
    @StartDate DATE,
    @EndDate DATE
AS
SELECT * FROM Orders
WHERE OrderDate BETWEEN @StartDate AND @EndDate

-- Решение: ветвление логики
CREATE PROCEDURE SearchOrders_Optimized
    @StartDate DATE,
    @EndDate DATE
AS
-- Если диапазон маленький (< 7 дней) - Index Seek
-- Если большой - Table Scan
DECLARE @DayDiff INT = DATEDIFF(DAY, @StartDate, @EndDate)

IF @DayDiff <= 7
    SELECT * FROM Orders
    WHERE OrderDate BETWEEN @StartDate AND @EndDate
    OPTION (RECOMPILE)  -- Для точной оценки
ELSE
    SELECT * FROM Orders WITH (INDEX(0))  -- Принудительный Table Scan
    WHERE OrderDate BETWEEN @StartDate AND @EndDate
```

### **Пример 2: Поиск с опциональными фильтрами**
```sql
-- Классическая проблема с OR
CREATE PROCEDURE FindCustomers
    @Name NVARCHAR(100) = NULL,
    @Email NVARCHAR(100) = NULL,
    @Phone VARCHAR(20) = NULL
AS
SELECT * FROM Customers
WHERE (@Name IS NULL OR CustomerName LIKE '%' + @Name + '%')
  AND (@Email IS NULL OR Email = @Email)
  AND (@Phone IS NULL OR Phone = @Phone)
OPTION (RECOMPILE)  -- Обязательно!

-- Без RECOMPILE: будет единый "универсальный" план (обычно плохой)
-- С RECOMPILE: оптимизатор видит фактические NULL/NOT NULL
```

### **Пример 3: Проблема с LIKE и разной селективностью**
```sql
CREATE PROCEDURE SearchProducts
    @SearchTerm NVARCHAR(100)
AS
-- Если @SearchTerm короткий (3 символа) → много результатов → Table Scan
-- Если @SearchTerm длинный (20 символов) → мало результатов → Index Seek

-- Решение с динамическим порогом
DECLARE @ResultEstimate INT

-- Оцениваем селективность по статистике
SELECT @ResultEstimate = histogram.rows 
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_histogram(s.object_id, s.stats_id) histogram
WHERE s.name = 'IX_Products_Name'
  AND histogram.range_high_key = LEFT(@SearchTerm, 4)  -- Примерно

IF @ResultEstimate > 10000  -- Порог
    SELECT * FROM Products 
    WHERE ProductName LIKE '%' + @SearchTerm + '%'
    OPTION (USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION'))
ELSE
    SELECT * FROM Products 
    WHERE ProductName LIKE '%' + @SearchTerm + '%'
    OPTION (RECOMPILE)
```

## **Мониторинг Parameter Sniffing**

```sql
-- Найти процедуры с potential sniffing проблемами
SELECT 
    OBJECT_NAME(ps.object_id) AS procedure_name,
    ps.execution_count,
    ps.total_worker_time / 1000 AS total_cpu_ms,
    ps.total_elapsed_time / 1000 AS total_duration_ms,
    ps.total_logical_reads,
    -- Разница между худшим и лучшим временем
    (ps.max_worker_time - ps.min_worker_time) / 1000 AS cpu_variance_ms,
    (ps.max_elapsed_time - ps.min_elapsed_time) / 1000 AS duration_variance_ms,
    -- Коэффициент вариации (> 2.0 - возможен sniffing)
    CAST((ps.max_worker_time - ps.min_worker_time) AS FLOAT) / 
    NULLIF(ps.total_worker_time / ps.execution_count, 0) AS cpu_variation_ratio
FROM sys.dm_exec_procedure_stats ps
WHERE ps.database_id = DB_ID()
    AND ps.execution_count > 100
    AND (ps.max_worker_time - ps.min_worker_time) > ps.total_worker_time / ps.execution_count
ORDER BY cpu_variation_ratio DESC

-- Смотреть кэшированные планы с разными параметрами
SELECT 
    cp.plan_handle,
    cp.objtype,
    cp.usecounts,
    st.text,
    qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE cp.objtype = 'Proc'
    AND st.text LIKE '%GetOrdersByDate%'
```

## **Правила работы с Parameter Sniffing**

### **Когда это ХОРОШО (нужно сохранить)**
```sql
-- Стабильные параметры, типичный паттерн
CREATE PROCEDURE GetOrderDetails @OrderID INT
AS
SELECT * FROM Orders WHERE OrderID = @OrderID
-- Всегда 1 строка, sniffing помогает!
```

### **Когда это ПЛОХО (нужно исправить)**
```sql
-- Параметры с разной селективностью
CREATE PROCEDURE GetOrders
    @Status VARCHAR(20),  -- 'New'(1%), 'Completed'(90%), 'Cancelled'(5%)
    @DateRange INT        -- 1(день), 7(неделя), 30(месяц)
-- Нужен RECOMPILE или другая стратегия
```

## **Автопараметризация и её проблемы**

```sql
-- Простой запрос (автопараметризуется)
SELECT * FROM Orders WHERE OrderID = 123
-- Становится: SELECT * FROM Orders WHERE OrderID = @1

-- Сложный запрос (не параметризуется)
SELECT * FROM Orders WHERE OrderDate > '2024-01-01' AND CustomerID = 456
-- Остается как есть → новый план для каждой даты!

-- Принудительная параметризация на уровне БД
ALTER DATABASE MyDB SET PARAMETERIZATION FORCED

-- Теперь даже сложные запросы параметризуются
SELECT * FROM Orders WHERE OrderDate > '2024-01-01' AND CustomerID = 456
-- Становится: WHERE OrderDate > @1 AND CustomerID = @2
```

### **Проблемы forced parameterization:**
1. **Loss of specificity**: `WHERE Price > 100` и `WHERE Price > 1000` → одинаковый план
2. **Constant folding issues**: вычисления с константами не оптимизируются
3. **Plan cache bloat**: много похожих планов

## **Лучшие практики**

### **1. Проектирование с учетом sniffing**
```sql
-- Заранее продумывать стратегию
CREATE PROCEDURE GetData
    @Param INT,
    @ExpectedRows INT = NULL  -- Опциональный hint от приложения
AS
IF @ExpectedRows > 10000
    SELECT * FROM LargeTable WITH (INDEX(0)) WHERE Column = @Param
ELSE
    SELECT * FROM LargeTable WHERE Column = @Param
```

### **2. Использование Query Store (SQL Server 2016+)**
```sql
-- Включить Query Store
ALTER DATABASE MyDB SET QUERY_STORE = ON

-- Принудительно использовать лучший план
EXEC sp_query_store_force_plan @query_id, @plan_id

-- Найти регрессии
SELECT 
    qsp.plan_id,
    qsp.query_id,
    qsp.avg_duration,
    qsp.avg_cpu_time,
    qt.query_sql_text
FROM sys.query_store_plan qsp
JOIN sys.query_store_query qsq ON qsp.query_id = qsq.query_id
JOIN sys.query_store_query_text qt ON qsq.query_text_id = qt.query_text_id
WHERE qsp.avg_duration > 1000  -- > 1 секунды
ORDER BY qsp.avg_duration DESC
```

### **3. Статистика и индексы**
```sql
-- Обновлять статистику для ключевых таблиц
UPDATE STATISTICS Orders WITH FULLSCAN

-- Создавать индексы, учитывая разные сценарии
CREATE INDEX IX_Orders_Date_Filtered ON Orders(OrderDate)
WHERE OrderDate > '2023-01-01'  -- Фильтрованный индекс для свежих данных
```

## **Decision Tree для выбора стратегии**

```
Есть проблема с Parameter Sniffing?
    │
    ├─ Запрос выполняется редко (< 10/час)? → OPTION (RECOMPILE)
    │
    ├─ Есть явно "типичный" сценарий? → OPTIMIZE FOR (@typical_value)
    │
    ├─ Параметры непредсказуемы? → OPTIMIZE FOR UNKNOWN
    │
    ├─ Можно разделить логику? → Разные процедуры для разных сценариев
    │
    ├─ SQL Server 2016+? → Query Store + принудительные планы
    │
    └─ Иначе → Локальные переменные (как последнее средство)
```

## **Каверзный вопрос на собеседовании**

**"Как бы вы объяснили Parameter Sniffing менеджеру?"**

> "Представьте, что у вас есть склад. Первый клиент приходит и просит одну коробку (план: сотрудник с тележкой). Мы обучаем сотрудника этому сценарию. Потом приходит клиент за 1000 коробок (нужен погрузчик), но сотрудник всё равно берет тележку, потому что его так научили. Parameter Sniffing — это когда SQL Server 'запоминает' первый сценарий и применяет его ко всем остальным, даже когда это неэффективно."

**Главное правило:** Parameter Sniffing — это не баг, а фича. Он помогает в 90% случаев. Проблемы возникают в 10% случаев с нетипичными параметрами. Лечите точечно, а не отключайте везде.
