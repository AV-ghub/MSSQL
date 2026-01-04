## **Что такое RECOMPILE на самом деле?**

> **RECOMPILE заставляет SQL Server перекомпилировать план запроса КАЖДЫЙ РАЗ при выполнении, используя актуальные значения параметров и статистику.**

### **Как работает план запроса без RECOMPILE**

```sql
-- 1. Первое выполнение: компиляция плана
CREATE PROCEDURE GetOrders @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate > @StartDate
-- Допустим, @StartDate = '2024-01-01' → 100 строк

-- 2. План кэшируется
-- 3. Второе выполнение: используется кэшированный план
EXEC GetOrders @StartDate = '2023-01-01'  -- 100,000 строк!
-- Используется тот же план, оптимизированный для 100 строк
-- Результат: Table Scan вместо Seek, медленно!
```

### **С RECOMPILE**
```sql
CREATE PROCEDURE GetOrders @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate > @StartDate
OPTION (RECOMPILE)  -- Перекомпиляция каждый раз!

-- 1. @StartDate = '2024-01-01' → 100 строк → Index Seek
-- 2. @StartDate = '2023-01-01' → 100,000 строк → Table Scan (правильно!)
-- 3. @StartDate = '2024-06-01' → 10 строк → Index Seek
```

## **Когда RECOMPILE полезен (реальные кейсы)**

### **1. Parameter Sniffing (нюхание параметров)**
```sql
-- Проблема: первый параметр определяет план для всех
-- Допустим, есть индекс по OrderDate
CREATE INDEX IX_Orders_Date ON Orders(OrderDate)

-- Процедура без RECOMPILE
CREATE PROCEDURE GetRecentOrders 
    @DaysBack INT
AS
SELECT * FROM Orders 
WHERE OrderDate > DATEADD(DAY, -@DaysBack, GETDATE())

-- Проблема:
EXEC GetRecentOrders @DaysBack = 1  -- 100 строк → Index Seek (план кэшируется)
EXEC GetRecentOrders @DaysBack = 365  -- 1,000,000 строк → Должен быть Table Scan, но будет Seek!
```

### **2. Динамические условия поиска**
```sql
-- Сложные фильтры с OR
CREATE PROCEDURE SearchOrders
    @OrderID INT = NULL,
    @CustomerID INT = NULL,
    @StartDate DATE = NULL,
    @EndDate DATE = NULL
AS
SELECT * FROM Orders
WHERE (@OrderID IS NULL OR OrderID = @OrderID)
  AND (@CustomerID IS NULL OR CustomerID = @CustomerID)
  AND (@StartDate IS NULL OR OrderDate >= @StartDate)
  AND (@EndDate IS NULL OR OrderDate <= @EndDate)
OPTION (RECOMPILE)  -- Нужен, т.к. план зависит от NULL/NOT NULL параметров

-- Без RECOMPILE: будет один универсальный (плохой) план
-- С RECOMPILE: оптимизатор "видит" фактические значения NULL/NOT NULL
```

### **3. Временные таблицы в процедурах**
```sql
CREATE PROCEDURE ComplexReport @Year INT
AS
-- Временная таблица с данными
CREATE TABLE #TempData (ID INT, Value DECIMAL(10,2))
INSERT INTO #TempData 
SELECT OrderID, TotalAmount 
FROM Orders 
WHERE YEAR(OrderDate) = @Year

-- Проблема: статистика по #TempData собирается пустая
-- При первом вызове: 0 строк → неверные оценки для JOIN
SELECT * 
FROM #TempData t
JOIN OrderDetails od ON t.ID = od.OrderID
OPTION (RECOMPILE)  -- Статистика пересчитывается при каждом вызове
```

## **Стоимость RECOMPILE: компромисс**

```sql
-- Измеряем стоимость компиляции
SET STATISTICS TIME ON

-- Без RECOMPILE (быстрое выполнение, но возможно плохой план)
EXEC GetOrders @StartDate = '2024-01-01'
-- CompileTime: 5ms, ExecutionTime: 10ms, Total: 15ms

-- С RECOMPILE (компиляция каждый раз)
EXEC GetOrders_Recompile @StartDate = '2024-01-01'  
-- CompileTime: 5ms, ExecutionTime: 10ms, Total: 15ms
-- Кажется одинаково? Но при 1000 вызовов:
-- Без RECOMPILE: 5 + (10 × 1000) = 10005ms
-- С RECOMPILE: (5 + 10) × 1000 = 15000ms (на 50% медленнее!)
```

## **Когда использовать RECOMPILE**

### **✅ Использовать когда:**
1. **Параметры сильно меняют селективность**
   ```sql
   -- 1% данных vs 99% данных
   CREATE PROCEDURE GetData @FilterType INT
   AS
   SELECT * FROM LargeTable
   WHERE (@FilterType = 1 AND ColumnA = 1)  -- 1% строк
      OR (@FilterType = 2 AND ColumnB = 1)  -- 99% строк
   OPTION (RECOMPILE)  -- Нужен разный план!
   ```

2. **Динамический SQL с переменным числом условий**
   ```sql
   CREATE PROCEDURE DynamicSearch @SearchParams XML
   AS
   DECLARE @SQL NVARCHAR(MAX)
   -- Построение динамического SQL...
   EXEC sp_executesql @SQL OPTION (RECOMPILE)
   ```

3. **Процедуры, вызываемые редко (раз в день/час)**
   ```sql
   -- Ежедневный отчет
   CREATE PROCEDURE DailyReport @ReportDate DATE
   AS
   -- Сложные агрегации
   OPTION (RECOMPILE)  -- Компиляция 1 раз в день - нормально
   ```

4. **Когда статистика часто устаревает**
   ```sql
   -- Таблица с частыми вставками
   CREATE PROCEDURE GetLatestOrders
   AS
   SELECT * FROM Orders 
   WHERE OrderDate > DATEADD(HOUR, -1, GETDATE())
   OPTION (RECOMPILE)  -- Актуальная статистика для новых данных
   ```

### **❌ НЕ использовать когда:**
1. **Часто вызываемые процедуры (1000+ вызовов в минуту)**
   ```sql
   -- Процедура авторизации (вызывается постоянно)
   CREATE PROCEDURE AuthenticateUser @Login VARCHAR(50)
   AS
   SELECT UserID FROM Users WHERE Login = @Login
   -- OPTION (RECOMPILE)  -- Будет дорого! План стабильный
   ```

2. **Простые запросы с постоянной селективностью**
   ```sql
   SELECT * FROM Products WHERE ProductID = @ID  -- Всегда 1 строка
   -- RECOMPILE не нужен
   ```

3. **Внутри циклов**
   ```sql
   WHILE @i < 100
   BEGIN
       EXEC SomeProc OPTION (RECOMPILE)  -- 100 компиляций!
       SET @i += 1
   END
   ```

## **Альтернативы RECOMPILE**

### **1. RECOMPILE на уровне процедуры**
```sql
-- Перекомпиляция всей процедуры
CREATE PROCEDURE GetOrders @StartDate DATE
WITH RECOMPILE  -- Весь текст процедуры перекомпилируется
AS
SELECT * FROM Orders WHERE OrderDate > @StartDate
-- Минус: перекомпилируются ВСЕ запросы в процедуре
```

### **2. OPTIMIZE FOR hint**
```sql
-- Подсказка оптимизатору
CREATE PROCEDURE GetOrders @StartDate DATE
AS
SELECT * FROM Orders WHERE OrderDate > @StartDate
OPTION (OPTIMIZE FOR (@StartDate = '2024-01-01'))
-- План оптимизируется под конкретное значение
-- Хорошо, если есть "типичный" сценарий
```

### **3. OPTIMIZE FOR UNKNOWN**
```sql
-- Использовать усредненные статистики
SELECT * FROM Orders WHERE OrderDate > @StartDate
OPTION (OPTIMIZE FOR UNKNOWN)
-- План не зависит от конкретного параметра
-- Среднее между всеми возможными значениями
```

### **4. Локальные переменные (старая техника)**
```sql
-- Обход parameter sniffing
CREATE PROCEDURE GetOrders @StartDate DATE
AS
DECLARE @LocalDate DATE = @StartDate
SELECT * FROM Orders WHERE OrderDate > @LocalDate
-- Оптимизатор не знает значение @LocalDate
-- Использует усредненную статистику
```

### **5. RECOMPILE только для отдельных запросов**
```sql
CREATE PROCEDURE ComplexProc
AS
-- Запрос 1 (стабильный)
SELECT * FROM ConfigTable  -- Без RECOMPILE

-- Запрос 2 (чувствительный к параметрам)
SELECT * FROM Orders WHERE Date > @Date 
OPTION (RECOMPILE)  -- Только этот запрос

-- Запрос 3 (стабильный)
SELECT * FROM Users WHERE Active = 1  -- Без RECOMPILE
```

## **Практические примеры**

### **Пример 1: Отчет с переменными фильтрами**
```sql
CREATE PROCEDURE SalesReport
    @RegionID INT = NULL,
    @ProductCategoryID INT = NULL,
    @StartDate DATE,
    @EndDate DATE
AS
-- Без RECOMPILE: будет искажать все NULL проверки как "неизвестно"
-- С RECOMPILE: оптимизатор видит реальные NULL/NOT NULL
SELECT 
    r.RegionName,
    p.ProductName,
    SUM(s.Quantity) AS TotalQuantity,
    SUM(s.Amount) AS TotalAmount
FROM Sales s
LEFT JOIN Regions r ON s.RegionID = r.RegionID
LEFT JOIN Products p ON s.ProductID = p.ProductID
WHERE (@RegionID IS NULL OR s.RegionID = @RegionID)
  AND (@ProductCategoryID IS NULL OR p.CategoryID = @ProductCategoryID)
  AND s.SaleDate BETWEEN @StartDate AND @EndDate
GROUP BY r.RegionName, p.ProductName
OPTION (RECOMPILE)  -- Критически важно!

-- Вызовы:
EXEC SalesReport NULL, NULL, '2024-01-01', '2024-12-31'  -- Полный отчет
EXEC SalesReport 1, NULL, '2024-01-01', '2024-12-31'     -- По региону
EXEC SalesReport NULL, 5, '2024-01-01', '2024-12-31'     -- По категории
-- Три РАЗНЫХ плана, каждый оптимальный
```

### **Пример 2: Проблема с TOP + ORDER BY**
```sql
CREATE PROCEDURE GetTopOrders
    @TopN INT,
    @SortColumn VARCHAR(50)
AS
DECLARE @SQL NVARCHAR(MAX)
SET @SQL = '
SELECT TOP (@TopN) * 
FROM Orders 
ORDER BY ' + QUOTENAME(@SortColumn)

-- Без RECOMPILE: план для первого вызова (например, @SortColumn = 'OrderDate')
-- С RECOMPILE: новый план для каждого столбца сортировки
EXEC sp_executesql @SQL, N'@TopN INT', @TopN
OPTION (RECOMPILE)
```

### **Пример 3: Мониторинг стоимости RECOMPILE**
```sql
-- Измеряем влияние
CREATE PROCEDURE TestRecompile @Param INT
AS
DECLARE @StartTime DATETIME2 = SYSDATETIME()

SELECT * FROM LargeTable 
WHERE Column1 = @Param
OPTION (RECOMPILE)  -- Или без

SELECT 
    DATEDIFF(MILLISECOND, @StartTime, SYSDATETIME()) AS DurationMs,
    @@ROWCOUNT AS RowsReturned
```

## **Как принимать решение: Decision Tree**

```
Нужен ли RECOMPILE?
    │
    ├─ Запрос выполняется редко (< 10 раз/час)? → ✅ ДА
    │
    ├─ Параметры сильно меняют селективность? → ✅ ДА  
    │   (1 строка vs 1M строк)
    │
    ├─ Много OR условий с NULL проверками? → ✅ ДА
    │
    ├─ Используются временные таблицы? → ✅ ДА
    │
    ├─ Запрос в цикле или часто вызывается? → ❌ НЕТ
    │
    ├─ План стабильный и эффективный? → ❌ НЕТ
    │
    └─ Можно использовать OPTIMIZE FOR? → ❌ НЕТ (использовать его)
```

## **Каверзные вопросы на собеседовании**

1. **"В чем разница между WITH RECOMPILE и OPTION (RECOMPILE)?"**
   - **Ответ**: 
     - `WITH RECOMPILE` — на уровне процедуры, перекомпилирует ВСЕ запросы в процедуре
     - `OPTION (RECOMPILE)` — на уровне запроса, только конкретный запрос
     ```sql
     CREATE PROC Test WITH RECOMPILE AS ...  -- Вся процедура
     CREATE PROC Test AS
     SELECT ... OPTION (RECOMPILE)  -- Только этот SELECT
     ```

2. **"Почему RECOMPILE не решает все проблемы с производительностью?"**
   - **Ответ**: Он добавляет overhead компиляции, не исправляет плохие индексы, не-SARGable условия или недостающую статистику.

3. **"Что такое 'проблема параметризации' и как RECOMPILE помогает?"**
   - **Ответ**: SQL Server может параметризовать запросы автоматически. RECOMPILE заставляет использовать фактические значения:
   ```sql
   SELECT * FROM Orders WHERE OrderID = 12345  -- Простой параметр
   -- Может стать: WHERE OrderID = @1
   -- RECOMPILE показывает реальное значение 12345
   ```

4. **"Можно ли использовать RECOMPILE в индексированных представлениях?"**
   - **Ответ**: Нет, для индексированных представлений план фиксированный.

## **Практическое правило**

> "Используйте `OPTION (RECOMPILE)` точечно для запросов, где **план должен меняться в зависимости от параметров**. Измеряйте overhead компиляции. Для часто вызываемых запросов ищите альтернативы: `OPTIMIZE FOR`, локальные переменные или перепроектирование логики."

**Сколько компиляций в секунду выдерживает SQL Server?** Примерно 100-200 компиляций/сек на среднем сервере. Если ваша процедура с RECOMPILE вызывается 1000 раз/сек — будет проблема.
