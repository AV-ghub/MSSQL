## **Вопрос 17: Проблемы с блокировками (deadlocks) и их решение**

Это один из самых сложных вопросов на собеседованиях, требующий понимания транзакций, изоляции и работы движка.

### **Короткий ответ для собеседования**
> "Deadlock — это ситуация, когда две или более транзакций блокируют ресурсы, которые нужны друг другу, и ни одна не может продолжить. SQL Server автоматически обнаруживает deadlock, выбирает 'жертву' (наименее затратную транзакцию) и откатывает её. Для предотвращения: 1) одинаковый порядок доступа к ресурсам, 2) короткие транзакции, 3) правильная изоляция, 4) индексы для быстрого поиска."

### **Как возникает deadlock**

```sql
-- Классический пример deadlock
-- Сессия 1
BEGIN TRANSACTION
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1
-- Блокирует строку AccountID = 1 (eXclusive lock)

-- Сессия 2 (параллельно)
BEGIN TRANSACTION  
UPDATE Accounts SET Balance = Balance + 200 WHERE AccountID = 2
-- Блокирует строку AccountID = 2 (X lock)

-- Теперь deadlock:
-- Сессия 1 пытается прочитать AccountID = 2 → ждет
-- Сессия 2 пытается прочитать AccountID = 1 → ждет
-- Взаимная блокировка!
```

### **Типы блокировок (locks)**

```sql
-- Основные типы:
-- 1. Shared (S) - для чтения
-- 2. Exclusive (X) - для записи (несовместима ни с чем)
-- 3. Update (U) - для будущего UPDATE
-- 4. Intent (IS, IX) - намерение заблокировать на уровне выше

-- Совместимость блокировок:
/*
         | S  | X  | U  | IS | IX |
---------|----|----|----|----|----|
S        | ✓  | ✗  | ✗  | ✓  | ✗  |
X        | ✗  | ✗  | ✗  | ✗  | ✗  |  
U        | ✗  | ✗  | ✗  | ✓  | ✗  |
IS       | ✓  | ✗  | ✓  | ✓  | ✓  |
IX       | ✗  | ✗  | ✗  | ✓  | ✓  |
*/
```

### **Как увидеть deadlock в логе**

```sql
-- Включить трассировку deadlock
DBCC TRACEON (1222, -1)  -- Вывод в error log
DBCC TRACEON (3605, -1)  -- В SQL Server log

-- Или Extended Events (рекомендуется)
CREATE EVENT SESSION [Deadlock_Monitor] ON SERVER 
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(SET filename=N'Deadlock_Monitor.xel')
GO

ALTER EVENT SESSION [Deadlock_Monitor] ON SERVER STATE = START
GO

-- Читаем deadlock graph
SELECT 
    XEvent.value('(data/value)[1]', 'varchar(max)') AS DeadlockGraph
FROM (
    SELECT CAST(event_data AS XML) AS XEventData
    FROM sys.fn_xe_file_target_read_file(
        'Deadlock_Monitor*.xel',
        NULL, NULL, NULL
    )
) AS Data
CROSS APPLY XEventData.nodes('//event[@name="xml_deadlock_report"]') AS XEvent(XEvent)
```

### **Стратегии предотвращения deadlock**

**1. Одинаковый порядок доступа**
```sql
-- ПЛОХО: Разный порядок в разных процедурах
-- Процедура 1: UPDATE Accounts... затем UPDATE Users...
-- Процедура 2: UPDATE Users... затем UPDATE Accounts...

-- ХОРОШО: Унифицированный порядок
-- Всегда: сначала Accounts, потом Users
CREATE PROCEDURE TransferMoney
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION
        
        -- Всегда сначала списываем
        UPDATE Accounts SET Balance = Balance - @Amount 
        WHERE AccountID = @FromAccount
        
        -- Затем зачисляем
        UPDATE Accounts SET Balance = Balance + @Amount
        WHERE AccountID = @ToAccount
        
        -- Затем другие таблицы
        INSERT INTO Transactions (...)
        VALUES (...)
        
        COMMIT TRANSACTION
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION
        THROW
    END CATCH
END
```

**2. Короткие транзакции**
```sql
-- ПЛОХО: Длинная транзакция
BEGIN TRANSACTION
-- Сложная бизнес-логика
-- Вызовы других процедур
-- Ожидание пользователя?!
COMMIT

-- ХОРОШО: Минимальная работа в транзакции
BEGIN TRANSACTION
    UPDATE Table1 SET ... WHERE ID = @ID
    -- ТОЛЬКО критичные операции
COMMIT

-- Остальная логика вне транзакции
EXEC LogActivity @ID
EXEC SendNotification @ID
```

**3. Правильная изоляция**
```sql
-- SNAPSHOT изоляция предотвращает многие deadlocks
ALTER DATABASE YourDB SET ALLOW_SNAPSHOT_ISOLATION ON

-- В сессии
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRANSACTION
SELECT * FROM Accounts  -- Чтение из snapshot
UPDATE Accounts SET ... -- Модификация
COMMIT
```

**4. Индексы для быстрого поиска**
```sql
-- Deadlock часто возникает при table/range scans
-- Без индекса: UPDATE WHERE Name = 'John' → Table Scan → X locks на всё
CREATE INDEX IX_Accounts_Name ON Accounts(Name)

-- С индексом: Seek → только нужные строки
```

### **Каверзные уточняющие вопросы**

1. **"Что такое deadlock priority и как его настроить?"**
   - **Ответ**: Приоритет транзакции при выборе жертвы:
   ```sql
   SET DEADLOCK_PRIORITY LOW    -- Первый кандидат на откат
   SET DEADLOCK_PRIORITY NORMAL -- По умолчанию
   SET DEADLOCK_PRIORITY HIGH   -- Последний кандидат
   SET DEADLOCK_PRIORITY @variable
   
   -- Числовые значения от -10 до 10
   SET DEADLOCK_PRIORITY -5
   ```

2. **"Можно ли полностью избежать deadlock?"**
   - **Ответ**: Нет, в конкурентных системах это невозможно. Но можно свести к минимуму. 1-2 deadlock'а в день на busy system — норма.

3. **"Что такое livelock?"**
   - **Ответ**: Ситуация, когда транзакции постоянно перезапускаются, но не прогрессируют. Реже в SQL Server.

4. **"Как обрабатывать deadlock на стороне приложения?"**
   - **Ответ**: Перезапускать транзакцию:
   ```csharp
   int retries = 3;
   for (int i = 0; i < retries; i++)
   {
       try 
       {
           ExecuteTransaction();
           break;
       }
       catch (SqlException ex) when (ex.Number == 1205) // Deadlock
       {
           if (i == retries - 1) throw;
           Thread.Sleep(100 * (i + 1)); // Экспоненциальная задержка
       }
   }
   ```

### **Практические примеры решения**

**Пример 1: Deadlock при обновлении в обратном порядке**
```sql
-- Проблема
-- Т1: UPDATE A → UPDATE B
-- Т2: UPDATE B → UPDATE A

-- Решение: хранимая процедура с унифицированным порядком
CREATE PROCEDURE UpdateRelatedTables 
    @ID INT,
    @Param1 VARCHAR(50),
    @Param2 VARCHAR(50)
AS
BEGIN
    DECLARE @DeadlockRetry TINYINT = 0
    
    WHILE @DeadlockRetry < 3
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION
            
            -- Всегда сначала TableA
            UPDATE TableA SET Column1 = @Param1 WHERE ID = @ID
            
            -- Затем TableB  
            UPDATE TableB SET Column2 = @Param2 WHERE A_ID = @ID
            
            COMMIT TRANSACTION
            BREAK
        END TRY
        BEGIN CATCH
            ROLLBACK TRANSACTION
            
            IF ERROR_NUMBER() = 1205 -- Deadlock
            BEGIN
                SET @DeadlockRetry += 1
                WAITFOR DELAY '00:00:00.100'
                CONTINUE
            END
            ELSE
                THROW
        END CATCH
    END
END
```

**Пример 2: Deadlock из-за lock escalation**
```sql
-- Проблема: много row locks → escalation до table lock
UPDATE Orders SET Status = 'Processed' WHERE CustomerID = 123
-- 10,000 строк → 10,000 row locks → Escalation → Table lock

-- Решение 1: Отключить escalation (осторожно!)
ALTER TABLE Orders SET (LOCK_ESCALATION = DISABLE)

-- Решение 2: Пакетная обработка
DECLARE @BatchSize INT = 1000
WHILE 1 = 1
BEGIN
    UPDATE TOP (@BatchSize) Orders 
    SET Status = 'Processed' 
    WHERE CustomerID = 123 AND Status = 'New'
    
    IF @@ROWCOUNT = 0 BREAK
    WAITFOR DELAY '00:00:00.010' -- Дать другим поработать
END
```

### **Мониторинг блокировок**

```sql
-- Текущие блокировки
SELECT 
    tl.request_session_id,
    wt.blocking_session_id,
    OBJECT_NAME(p.object_id) AS TableName,
    tl.resource_type,
    tl.request_mode,
    tl.request_status,
    wt.wait_duration_ms,
    t.text AS QueryText
FROM sys.dm_tran_locks tl
LEFT JOIN sys.dm_os_waiting_tasks wt ON tl.lock_owner_address = wt.resource_address
LEFT JOIN sys.partitions p ON tl.resource_associated_entity_id = p.hobt_id
LEFT JOIN sys.dm_exec_requests er ON tl.request_session_id = er.session_id
OUTER APPLY sys.dm_exec_sql_text(er.sql_handle) t
WHERE tl.resource_database_id = DB_ID()
ORDER BY tl.request_session_id

-- Поиск частых deadlocks
SELECT 
    COUNT(*) AS DeadlockCount,
    FORMAT(dt.event_date, 'yyyy-MM-dd') AS EventDate,
    -- Анализ графа deadlock
    -- (реальный анализ требует парсинга XML)
FROM (
    SELECT 
        DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), event_data.value('(@timestamp)[1]', 'datetime2')) AS event_date
    FROM sys.fn_xe_file_target_read_file('Deadlock_Monitor*.xel', NULL, NULL, NULL)
    CROSS APPLY (SELECT CAST(event_data AS XML) AS event_data) AS data
) dt
GROUP BY FORMAT(dt.event_date, 'yyyy-MM-dd')
ORDER BY EventDate DESC
```

---

## **Вопрос 18: Разница между временными таблицами и табличными переменными**

Этот вопрос проверяет понимание областей видимости, производительности и использования temp объектов.

### **Короткий ответ для собеседования**
> "Временные таблицы (#temp) создаются в tempdb, имеют статистику, поддерживают индексы и могут быть видны в дочерних областях. Табличные переменные (@table) хранятся в памяти (до определенного размера), не имеют статистики, не поддерживают некластеризованные индексы (кроме PK/UNIQUE) и видны только в текущем пакете. Для небольших наборов данных (< 100 строк) — табличные переменные, для больших — временные таблицы."

### **Сравнение в таблице**

| Характеристика | Временная таблица (#temp) | Табличная переменная (@table) |
|----------------|--------------------------|-------------------------------|
| **Место хранения** | Tempdb (диск) | Memory (достаточная) → tempdb |
| **Статистика** | Да, автоматически | Нет (всегда 1 строка в оценке) |
| **Индексы** | Любые (после создания) | Только PK, UNIQUE (при объявлении) |
| **Область видимости** | Сессия/дочерние процедуры | Текущий пакет |
| **Транзакции** | Участвуют полностью | Только на момент модификации |
| **Параллелизм** | Да (если нужно) | Нет (serial) |
| **DDL (ALTER)** | Можно | Нельзя (только при объявлении) |
| **Внешние ключи** | Можно | Нельзя |
| **TRUNCATE** | Можно | Нельзя |

### **Создание и использование**

```sql
-- Временная таблица
CREATE TABLE #TempOrders (
    OrderID INT PRIMARY KEY,
    CustomerID INT,
    Amount DECIMAL(10,2),
    OrderDate DATE
)

-- Можно добавить индекс потом
CREATE INDEX IX_Temp_Customer ON #TempOrders(CustomerID)

-- Можно изменять
ALTER TABLE #TempOrders ADD Status VARCHAR(20)

-- Видна в дочерних процедурах
EXEC ChildProcedure  -- Может обращаться к #TempOrders

-- Автоматически удаляется при закрытии сессии
-- Или явно
DROP TABLE #TempOrders


-- Табличная переменная
DECLARE @TableVar TABLE (
    OrderID INT PRIMARY KEY,  -- PK можно
    CustomerID INT,
    Amount DECIMAL(10,2),
    OrderDate DATE
    -- INDEX IX_Customer (CustomerID) -- Только SQL Server 2014+
)

-- Нельзя добавить индекс после объявления
-- CREATE INDEX ... ON @TableVar -- ОШИБКА!

-- Нельзя ALTER
-- ALTER TABLE @TableVar ADD ... -- ОШИБКА!

-- Не видна в дочерних процедурах
EXEC ChildProcedure  -- НЕ видит @TableVar

-- Автоматически удаляется при завершении пакета
```

### **Производительность: тестовый пример**

```sql
-- Тест с 100K строк
CREATE TABLE SourceData (ID INT IDENTITY PRIMARY KEY, Data VARCHAR(100))
INSERT INTO SourceData 
SELECT TOP 100000 'Data_' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS VARCHAR)
FROM sys.all_objects a CROSS JOIN sys.all_objects b

SET STATISTICS IO, TIME ON

-- Тест 1: Табличная переменная (плохо для больших данных)
DECLARE @TableVar TABLE (ID INT, Data VARCHAR(100))

INSERT INTO @TableVar
SELECT ID, Data FROM SourceData
-- Оценка: 1 строка! План будет плохим

SELECT COUNT(*) FROM @TableVar
-- Table Scan, но оптимизатор думает что 1 строка

-- Тест 2: Временная таблица
CREATE TABLE #TempTable (ID INT, Data VARCHAR(100))

INSERT INTO #TempTable
SELECT ID, Data FROM SourceData
-- Сбор статистики

SELECT COUNT(*) FROM #TempTable
-- Правильный план, правильные оценки
```

### **Каверзные уточняющие вопросы**

1. **"Что такое 'tempdb spill' и когда происходит?"**
   - **Ответ**: Когда табличная переменная не помещается в память, данные сбрасываются в tempdb. Происходит при больших объемах или нехватке памяти.

2. **"Можно ли создать индекс на табличной переменной?"**
   - **Ответ**: С SQL Server 2014+ — да, но только при объявлении:
   ```sql
   DECLARE @TableVar TABLE (
       ID INT PRIMARY KEY,
       Data VARCHAR(100),
       INDEX IX_Data (Data)  -- SQL Server 2014+
   )
   ```

3. **"В чем разница между ##global_temp и #local_temp?"**
   - **Ответ**: ##global_temp видна всем сессиям, удаляется когда все сессии отключаются. #local_temp — только для текущей сессии.

4. **"Как выбрать между ними для table-valued параметров?"**
   - **Ответ**: TVP (Table-Valued Parameters) — отдельный механизм, похож на табличную переменную, но передается в процедуру.

### **Оптимизация с временными таблицами**

```sql
-- Вариант 1: С CREATE и DROP
CREATE TABLE #Temp (ID INT, Data VARCHAR(100))
-- работа
DROP TABLE #Temp

-- Вариант 2: Без DROP (автоочистка)
SELECT ID, Data INTO #Temp2 FROM SourceTable WHERE Condition = 1
-- Автоматически удалится при закрытии сессии

-- Вариант 3: TRUNCATE для повторного использования
CREATE TABLE #Temp3 (ID INT)
-- Первое использование
INSERT INTO #Temp3 SELECT ...
-- Очистка
TRUNCATE TABLE #Temp3
-- Второе использование
INSERT INTO #Temp3 SELECT ...
```

### **Особые случаи**

**Случай 1: Рекурсивные запросы с табличными переменными**
```sql
-- Иерархические данные
DECLARE @Hierarchy TABLE (
    ID INT,
    ParentID INT,
    Level INT
)

INSERT INTO @Hierarchy
SELECT ID, ParentID, 0 FROM Employees WHERE ManagerID IS NULL

WHILE @@ROWCOUNT > 0
BEGIN
    INSERT INTO @Hierarchy
    SELECT e.ID, e.ParentID, h.Level + 1
    FROM Employees e
    INNER JOIN @Hierarchy h ON e.ManagerID = h.ID
    WHERE NOT EXISTS (SELECT 1 FROM @Hierarchy WHERE ID = e.ID)
END
```

**Случай 2: Кэширование сложных вычислений**
```sql
-- Временная таблица как кэш
CREATE TABLE #Calculations (
    InputID INT PRIMARY KEY,
    ComplexResult DECIMAL(38,10),
    CalculationTime DATETIME
)

-- Заполняем сложными вычислениями
INSERT INTO #Calculations
SELECT 
    ID,
    dbo.ComplexFunction(ID, Param1, Param2) AS Result,
    GETDATE()
FROM InputTable

-- Многократное использование
SELECT * FROM MainTable m
INNER JOIN #Calculations c ON m.ID = c.InputID
WHERE c.ComplexResult > 1000
```

**Случай 3: Пакетная обработка с временными таблицами**
```sql
-- Обработка больших данных пакетами
CREATE TABLE #BatchProcessing (
    RowNum INT IDENTITY PRIMARY KEY,
    DataID INT,
    Processed BIT DEFAULT 0
)

INSERT INTO #BatchProcessing (DataID)
SELECT ID FROM VeryLargeTable

DECLARE @BatchSize INT = 1000
DECLARE @StartID INT, @EndID INT

WHILE 1 = 1
BEGIN
    SELECT TOP 1 @StartID = RowNum 
    FROM #BatchProcessing 
    WHERE Processed = 0 
    ORDER BY RowNum
    
    IF @StartID IS NULL BREAK
    
    SET @EndID = @StartID + @BatchSize - 1
    
    -- Обработка пакета
    UPDATE bp SET Processed = 1
    FROM #BatchProcessing bp
    WHERE bp.RowNum BETWEEN @StartID AND @EndID
      AND bp.Processed = 0
    
    -- Основная обработка
    EXEC ProcessBatch @StartID, @EndID
END
```

### **Практическое правило выбора**

```sql
-- ДЛЯ ТАБЛИЧНЫХ ПЕРЕМЕННЫХ:
-- 1. Мало строк (< 100-1000)
-- 2. Простые запросы (нет JOIN с большими таблицами)
-- 3. Нужна скорость создания/удаления
-- 4. Данные только для текущей логики

-- Пример:
DECLARE @UserIDs TABLE (UserID INT PRIMARY KEY)
INSERT INTO @UserIDs VALUES (1), (2), (3)
SELECT * FROM Users WHERE UserID IN (SELECT UserID FROM @UserIDs)


-- ДЛЯ ВРЕМЕННЫХ ТАБЛИЦ:
-- 1. Много строк (> 1000)
-- 2. Сложные JOIN/GROUP BY
-- 3. Нужна статистика для оптимизатора
-- 4. Индексы для производительности
-- 5. Нужно передать в дочерние процедуры

-- Пример:
CREATE TABLE #UserOrders (
    UserID INT,
    OrderCount INT,
    TotalAmount DECIMAL(10,2)
)

INSERT INTO #UserOrders
SELECT UserID, COUNT(*), SUM(Amount)
FROM Orders
GROUP BY UserID

CREATE INDEX IX_User ON #UserOrders(UserID)

EXEC AnalyzeUserOrders  -- Использует #UserOrders
```

### **Производительность в цифрах**

```sql
-- Тест с 1M строк
/*
Результаты (примерные):
                   | Время создания | Время запроса | Memory/Диск
Табличная переменная | 2 сек          | 15 сек         | Memory → tempdb spill
Временная таблица   | 3 сек          | 3 сек          | Tempdb с индексами

Выводы:
- До 1000 строк: табличные переменные быстрее
- После 1000: временные таблицы с индексами
- При JOIN с другими таблицами: временные таблицы всегда лучше
*/
```

---
