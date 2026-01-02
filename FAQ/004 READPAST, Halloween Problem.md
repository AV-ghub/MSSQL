## **Вопрос 8: Как работает READPAST hint и когда его использовать?**

### **Короткий ответ для собеседования**
> "READPAST — это hint, который заставляет SELECT или UPDATE пропускать строки, заблокированные другими транзакциями. Вместо ожидания разблокировки (как при обычном SELECT) или блокировки всего набора (как с UPDLOCK), запрос просто игнорирует заблокированные строки и работает с доступными. Используется для реализации очередей и high-concurrency систем, где потеря некоторых строк допустима."

### **Как работает READPAST на практике**

```sql
-- Сессия 1: Блокирует строки
BEGIN TRANSACTION
UPDATE QueueTable SET Status = 'Processing' WHERE ID = 1
-- НЕ коммитим пока!

-- Сессия 2: Обычный SELECT (ждет разблокировки)
SELECT * FROM QueueTable WHERE Status = 'Pending'
-- Будет ЖДАТЬ, пока сессия 1 не завершит транзакцию

-- Сессия 3: READPAST (пропускает заблокированное)
SELECT * FROM QueueTable WITH (READPAST) WHERE Status = 'Pending'
-- Немедленно возвращает результат, исключая строку с ID=1
```

### **Типичный use-case: очередь задач**

```sql
-- 1. Таблица очереди
CREATE TABLE TaskQueue (
    TaskID INT IDENTITY PRIMARY KEY,
    TaskData NVARCHAR(MAX),
    Status VARCHAR(20) DEFAULT 'Pending',
    WorkerID INT NULL,
    Created DATETIME DEFAULT GETDATE()
)

CREATE INDEX IX_TaskQueue_Status ON TaskQueue(Status) 
WHERE Status = 'Pending'

-- 2. Процедура получения задачи (многопоточная безопасная)
CREATE PROCEDURE GetNextTask
    @WorkerID INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TaskID INT
    
    BEGIN TRANSACTION
    
    -- Берём первую доступную задачу, пропуская заблокированные
    SELECT TOP 1 @TaskID = TaskID
    FROM TaskQueue WITH (READPAST)
    WHERE Status = 'Pending'
    ORDER BY TaskID
    
    -- Помечаем как обрабатываемую
    UPDATE TaskQueue 
    SET Status = 'Processing', 
        WorkerID = @WorkerID,
        Started = GETDATE()
    WHERE TaskID = @TaskID
    
    COMMIT TRANSACTION
    
    -- Возвращаем задачу
    SELECT * FROM TaskQueue WHERE TaskID = @TaskID
END
```

### **Разница между READPAST и другими hints**

```sql
-- Сравнение в таблице
/*
| Hint         | Заблокированные строки | Блокировка других | Использование |
|--------------|------------------------|-------------------|---------------|
| (READPAST)   | Пропускает             | Не блокирует      | Очереди       |
| (NOLOCK)     | Читает грязные данные  | Не блокирует      | Отчеты        |
| (UPDLOCK)    | Ждёт + блокирует на UPDATE | Блокирует на UPDATE | Сериализация |
| (XLOCK)      | Ждёт + эксклюзивная    | Эксклюзивная      | Критичные операции |
| (ROWLOCK)    | -                      | Только строки     | Уменьшение блокировок |
| (TABLOCK)    | -                      | Всю таблицу       | Массовые операции |
*/
```

### **Важные ограничения READPAST**

1. **Только для READ COMMITTED и REPEATABLE READ**
   ```sql
   SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
   SELECT * FROM Table WITH (READPAST) -- НЕ РАБОТАЕТ!
   ```

2. **Работает только со строковыми блокировками**
   ```sql
   -- Если таблица заблокирована целиком (TABLOCK)
   BEGIN TRANSACTION
   SELECT * FROM QueueTable WITH (TABLOCK)
   
   -- READPAST не поможет, будет ждать
   SELECT * FROM QueueTable WITH (READPAST) -- ЖДЁТ!
   ```

3. **Не работает с page locks**
   - Если установлен LOCK_ESCALATION = TABLE

4. **Только для SELECT и UPDATE**
   ```sql
   DELETE FROM Table WITH (READPAST) -- Не работает
   INSERT INTO Table WITH (READPAST) -- Не работает
   ```

### **Каверзные уточняющие вопросы**

1. **"Что будет, если все строки заблокированы?"**
   - **Ответ**: READPAST вернёт пустой результат, а не будет ждать.

2. **"Можно ли использовать READPAST для обновления?"**
   - **Ответ**: Да, но с синтаксисом `UPDATE ... WITH (READPAST)`:
   ```sql
   UPDATE TOP (1) QueueTable WITH (READPAST)
   SET Status = 'Processing'
   WHERE Status = 'Pending'
   ```

3. **"Чем READPAST лучше обработки deadlock'ов?"**
   - **Ответ**: READPAST предотвращает блокировки, а не обрабатывает их последствия. Меньше накладных расходов.

4. **"Как обеспечить fairness в очереди с READPAST?"**
   - **Ответ**: Нужен `ORDER BY` + возможно `SKIP LOCKED` (в SQL Server 2019+):
   ```sql
   -- SQL Server 2019+
   DELETE FROM QueueTable WITH (READPAST, SKIP_LOCKED)
   OUTPUT deleted.*
   WHERE Status = 'Pending'
   ORDER BY TaskID
   ```

### **Реальный пример: система обработки заказов**

```sql
-- Несколько worker'ов обрабатывают заказы
WHILE 1 = 1
BEGIN
    DECLARE @OrderID INT
    
    BEGIN TRY
        BEGIN TRANSACTION
        
        -- Берём следующий заказ, пропуская заблокированные
        SELECT TOP 1 @OrderID = OrderID
        FROM Orders WITH (READPAST)
        WHERE Status = 'New'
        ORDER BY CreatedDate
        
        IF @OrderID IS NOT NULL
        BEGIN
            UPDATE Orders 
            SET Status = 'Processing',
                Worker = SYSTEM_USER,
                Started = GETDATE()
            WHERE OrderID = @OrderID
        END
        
        COMMIT TRANSACTION
        
        IF @OrderID IS NOT NULL
        BEGIN
            -- Обработка заказа
            EXEC ProcessOrder @OrderID
            
            UPDATE Orders 
            SET Status = 'Completed',
                Finished = GETDATE()
            WHERE OrderID = @OrderID
        END
        ELSE
        BEGIN
            -- Нет задач - ждём
            WAITFOR DELAY '00:00:01'
        END
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION
        -- Логирование ошибки
    END CATCH
END
```

---

## **Вопрос 9: Что такое Halloween Problem (Проблема Хэллоуина)?**

### **Короткий ответ**
> "Halloween Problem — это ситуация, когда обновление строк через индекс может привести к их повторной обработке. SQL Server перемещается по индексу в определённом порядке, и если UPDATE изменяет значение ключевого столбца так, что строка 'перепрыгивает' вперёд по порядку индекса, она может быть обработана повторно. SQL Server решает эту проблему через Split/Sort/Collapse или использование временных структур."

### **Классический пример**

```sql
-- Таблица с индексом по зарплате
CREATE TABLE Employees (
    ID INT PRIMARY KEY,
    Name VARCHAR(50),
    Salary DECIMAL(10, 2)
)

CREATE INDEX IX_Salary ON Employees(Salary)

-- Вставляем тестовые данные
INSERT INTO Employees VALUES 
(1, 'Alice', 1000),
(2, 'Bob', 2000),
(3, 'Charlie', 3000)

-- Halloween Problem проявляется здесь:
UPDATE Employees 
SET Salary = Salary * 1.1  -- Увеличиваем всем зарплату на 10%
WHERE Salary < 2500

-- Что происходит НА САМОМ ДЕЛЕ:
-- 1. Читаем Alice (Salary = 1000) → обновляем на 1100
-- 2. Читаем Bob (Salary = 2000) → обновляем на 2200
-- 3. Читаем... а Alice теперь имеет Salary = 1100, 
--    но она уже прошла по порядку индекса!
-- Без защиты Alice могла бы получить 1100 * 1.1 = 1210
```

### **Как SQL Server защищается**

**План запроса покажет защиту:**

```sql
-- Смотрим план
SET SHOWPLAN_TEXT ON
UPDATE Employees SET Salary = Salary * 1.1 WHERE Salary < 2500

/*
|-- Compute Scalar (вычисление нового Salary)
|-- Split/Sort/Collapse (защита от Halloween)
|-- Clustered Index Update (обновление)
|-- Index Update (IX_Salary)
|-- Index Seek (IX_Salary) - поиск по WHERE
*/
```

### **Различные способы защиты**

1. **Split/Sort/Collapse (наиболее частый)**
   - Split: разделяет операции на DELETE и INSERT
   - Sort: сортирует по ключу
   - Collapse: объединяет операции

2. **Table Spool (Eager Spool)**
   ```sql
   -- SQL Server создает временную копию данных
   UPDATE Table SET Col = Col + 1
   -- План: Table Spool → обновление из спула
   ```

3. **Использование другого индекса**
   - Если есть несколько индексов, оптимизатор может выбрать неупорядоченный доступ

### **Когда возникает проблема**

1. **Обновление ключевого столбца индекса**
   ```sql
   UPDATE T SET IndexedColumn = IndexedColumn + 1
   ```

2. **Обновление, меняющее порядок в индексе**
   ```sql
   -- Даже если не ключевой столбец, но влияет на порядок
   UPDATE T SET Col = Col + 1 WHERE Col < 100
   -- Строки могут 'перепрыгнуть' через границу 100
   ```

3. **Удаление с переносом**
   ```sql
   DELETE FROM Source OUTPUT deleted.* INTO Target
   -- Если Target имеет тот же индекс
   ```

### **Как обнаружить в плане запроса**

```sql
-- Ищем признаки в XML плане
SELECT 
    qp.query_plan.value('declare default element namespace 
        "http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        count(//RelOp[@PhysicalOp="Sort"]/../..)', 'int') AS HasSort,
    qp.query_plan.value('declare default element namespace 
        "http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        count(//RelOp[@PhysicalOp="Table Spool"])', 'int') AS HasSpool
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE cp.objtype = 'Proc'
    AND qp.query_plan.exist('declare default element namespace 
        "http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //StmtSimple[@StatementType="UPDATE"]') = 1
```

### **Каверзные уточняющие вопросы**

1. **"Почему проблема называется Halloween?"**
   - **Ответ**: Обнаружена 31 октября 1976 года исследователями IBM при работе над System R.

2. **"Всегда ли Split/Sort/Collapse замедляет запрос?"**
   - **Ответ**: Да, это дополнительная работа. Но без неё запрос даст неверный результат.

3. **"Можно ли отключить защиту?"**
   - **Ответ**: Нет, это фундаментальная гарантия корректности. Но можно изменить запрос:
   ```sql
   -- Медленно, с защитой
   UPDATE Employees SET Salary = Salary * 1.1 WHERE Salary < 2500
   
   -- Быстрее, разными путями
   -- Вариант 1: Через временную таблицу
   SELECT ID, Salary * 1.1 AS NewSalary 
   INTO #Temp 
   FROM Employees WHERE Salary < 2500
   
   UPDATE e SET e.Salary = t.NewSalary
   FROM Employees e 
   INNER JOIN #Temp t ON e.ID = t.ID
   
   -- Вариант 2: Если ID кластеризованный
   UPDATE e SET Salary = Salary * 1.1
   FROM Employees e 
   WHERE EXISTS (
       SELECT 1 FROM Employees e2 
       WHERE e2.ID = e.ID AND e2.Salary < 2500
   )
   ```

4. **"Влияет ли на производительность?"**
   - **Ответ**: Значительно! Split/Sort/Collapse может увеличить стоимость в 2-3 раза:
   ```
   Без защиты: Index Seek → Update
   С защитой: Index Seek → Split → Sort → Collapse → Update
   ```

### **Практический пример с демонстрацией**

```sql
-- Создаем тестовую таблицу
CREATE TABLE HalloweenTest (
    ID INT IDENTITY PRIMARY KEY,
    Value INT,
    GroupID INT
)

CREATE INDEX IX_Value ON HalloweenTest(Value)

-- Наполняем данными
INSERT INTO HalloweenTest (Value, GroupID)
SELECT 
    ROW_NUMBER() OVER (ORDER BY (SELECT NULL)),
    NTILE(10) OVER (ORDER BY (SELECT NULL))
FROM sys.all_columns
WHERE object_id < 100

-- Запрос, вызывающий проблему
UPDATE HalloweenTest 
SET Value = Value + 1000 
WHERE Value < 500

-- Смотрим план - увидим Split/Sort/Collapse
SET STATISTICS PROFILE ON
UPDATE HalloweenTest SET Value = Value + 1000 WHERE Value < 500
SET STATISTICS PROFILE OFF

-- Альтернатива без Split/Sort
UPDATE h
SET Value = h.Value + 1000
FROM HalloweenTest h
INNER JOIN (
    SELECT ID FROM HalloweenTest 
    WHERE Value < 500
) src ON h.ID = src.ID
-- Проверьте план - Split/Sort/Collapse может исчезнуть!
```

### **Как избегать Halloween Problem**

1. **Не обновлять ключевые столбцы индексов**
2. **Использовать промежуточные таблицы**
3. **Обновлять по кластеризованному ключу**
4. **Разделять на несколько запросов**

---
