## **Вопрос 2: Что такое NOLOCK и когда его использовать? Почему его называют "грязным чтением"?**

Это один из самых популярных и опасных вопросов на собеседованиях.

### **Короткий ответ для собеседования**
> "NOLOCK — это hint (указание), который отключает блокировки shared locks при чтении данных. Это позволяет читать незафиксированные данные (грязное чтение), пропускать строки (неповторяемое чтение) и видеть фантомные строки. Использовать его можно только в отчетных системах, где нетребовательна точность данных в реальном времени."

### **Развернутый разбор**

```sql
-- Как используется
SELECT * FROM Orders WITH (NOLOCK) WHERE Status = 'Shipped'
-- Эквивалентно
SELECT * FROM Orders (READUNCOMMITTED)
```

**Что происходит на самом деле:**
1. **Читаются незакоммиченные данные** (Dirty Reads)
2. **Возможны пропуски строк** (Non-repeatable Reads)
3. **Появление "фантомных" строк** (Phantom Reads)
4. **Ошибка "деление на ноль"** даже если данных нет!

### **Конкретный опасный пример**
```sql
-- Сессия 1: Начало транзакции, но НЕ коммит
BEGIN TRANSACTION
UPDATE Users SET Balance = Balance - 100 WHERE UserId = 1
-- Balance стал 900, но транзакция не завершена!

-- Сессия 2: Чтение с NOLOCK
SELECT Balance FROM Users WITH (NOLOCK) WHERE UserId = 1
-- Результат: 900 (незакоммиченные данные!)

-- Сессия 1: ROLLBACK (откат)
ROLLBACK
-- Баланс снова 1000, но вторая сессия уже увидела 900!
```

### **Когда можно использовать NOLOCK?**
Только если выполняются **ВСЕ** условия:
- **Система отчетности** (не операционная)
- **Данные устаревают на 1-2 минуты** — не критично
- **Нет транзакций UPDATE/DELETE** во время чтения
- **Разработчик понимает все риски**

### **Лучшие альтернативы NOLOCK**

```sql
-- 1. SNAPSHOT ISOLATION (рекомендуется)
ALTER DATABASE YourDB SET ALLOW_SNAPSHOT_ISOLATION ON
-- Затем в сессии:
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
SELECT * FROM Orders WHERE Status = 'Shipped'

-- 2. READ COMMITTED SNAPSHOT
ALTER DATABASE YourDB SET READ_COMMITTED_SNAPSHOT ON
-- Теперь READ COMMITTED работает через snapshot

-- 3. Columnstore индексы для отчетов
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Orders ON Orders

-- 4. Реплика на отчетный сервер
```

### **Каверзные уточняющие вопросы**

1. **"Почему NOLOCK не ускоряет запросы?"**
   - **Ответ**: Он не ускоряет сам запрос, а уменьшает блокировки, что может уменьшить ожидания при высокой конкурентности. Но может замедлить из-за rollback при ошибках.

2. **"В чем разница между NOLOCK и READ UNCOMMITTED?"**
   - **Ответ**: Это синонимы на уровне изоляции, но `NOLOCK` — это hint для конкретной таблицы, а `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` — для всей сессии.

3. **"Можно ли использовать NOLOCK в INSERT/UPDATE?"**
   - **Ответ**: Нет, только в SELECT. Для модификации есть другие hints (UPDLOCK, XLOCK и т.д.).

### **Как отвечать на собеседовании**
> "Я понимаю, что NOLOCK позволяет читать незакоммиченные данные. В production его использовать не рекомендую. Вместо этого настройку бы SNAPSHOT ISOLATION или создал колоночное хранилище для отчетных запросов."

---

## **Вопрос 3: Как найти самые тяжелые запросы по CPU/IO?**

Это проверка знания динамических административных представлений (DMV).

### **Правильный ответ**
```sql
-- Топ-10 запросов по CPU
SELECT TOP 10
    qs.execution_count,
    qs.total_worker_time / 1000 AS total_cpu_ms,
    qs.total_worker_time / qs.execution_count / 1000 AS avg_cpu_ms,
    SUBSTRING(st.text, 
              (qs.statement_start_offset/2) + 1,
              ((CASE qs.statement_end_offset 
                WHEN -1 THEN DATALENGTH(st.text)
                ELSE qs.statement_end_offset 
                END - qs.statement_start_offset)/2) + 1) AS query_text,
    DB_NAME(st.dbid) AS database_name,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC

-- Топ-10 по чтениям
SELECT TOP 10
    qs.execution_count,
    qs.total_logical_reads + qs.total_logical_writes AS total_io,
    qs.total_logical_reads,
    qs.total_logical_writes,
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1, ...) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_io DESC
```

### **Каверзные уточнения**

1. **"А если сервер только что перезапустили, в DMV нет данных. Что делать?"**
   - **Ответ**: Использовать Extended Events или SQL Server Profiler для сбора статистики, либо Query Store (если SQL 2016+).

2. **"В чем разница между logical reads и physical reads?"**
   - **Ответ**: Logical reads — чтение из cache (буферный пул), physical reads — с диска. Большое количество physical reads указывает на нехватку памяти или отсутствие индексов.

3. **"Как найти запросы с implicit conversion?"**
   ```sql
   -- Ищем implicit conversion в планах запросов
   SELECT 
       qp.query_plan.value('declare default element namespace 
       "http://schemas.microsoft.com/sqlserver/2004/07/showplan";
       (//*:Convert/@Implied)[1]', 'int') AS has_implicit_conversion,
       st.text
   FROM sys.dm_exec_cached_plans cp
   CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
   CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
   WHERE qp.query_plan.exist('declare default element namespace 
   "http://schemas.microsoft.com/sqlserver/2004/07/showplan";
   //*:Convert[@Implied=1]') = 1
   ```

---

## **Вопрос 4: В чем разница между UNION и UNION ALL? Какой быстрее и почему?**

### **Короткий ответ**
> "UNION удаляет дубликаты, для этого требуется сортировка и сравнение данных. UNION ALL не удаляет дубликаты и работает быстрее, так как просто объединяет результаты. Если дубликаты не важны или их нет — всегда использовать UNION ALL."

### **Визуализация разницы**
```sql
-- Данные
SET 1: {1, 2, 3, 3, 4}
SET 2: {3, 4, 5, 5}

-- UNION: {1, 2, 3, 4, 5} (дубли удалены)
-- UNION ALL: {1, 2, 3, 3, 4, 3, 4, 5, 5} (все как есть)
```

### **Производительность (разница может быть в 10-100 раз!)**
```sql
-- Медленно: сортировка + удаление дублей
SELECT Col1 FROM Table1
UNION
SELECT Col1 FROM Table2

-- Быстро: простое объединение
SELECT Col1 FROM Table1
UNION ALL
SELECT Col1 FROM Table2
```

### **План выполнения**
- **UNION ALL**: Concatenation operator → готово
- **UNION**: Concatenation → Sort/Distinct Sort → готово

### **Каверзные уточнения**

1. **"А если сделать UNION с DISTINCT — будет то же самое?"**
   - **Ответ**: Да, `SELECT ... UNION SELECT ...` эквивалентно `SELECT DISTINCT ... FROM (SELECT ... UNION ALL SELECT ...)`.

2. **"Когда UNION может быть быстрее UNION ALL?"**
   - **Ответ**: Практически никогда. Разве что если данные уже отсортированы и нужны уникальные значения, но и тогда разница минимальна.

3. **"Как объединить более 2 выборок с разной обработкой дублей?"**
   ```sql
   -- Разные комбинации
   SELECT * FROM A
   UNION ALL          -- Не удаляем дубли между A и B
   SELECT * FROM B
   UNION              -- Удаляем дубли между (A+B) и C
   SELECT * FROM C
   ```

---

## **Вопрос 5: Что такое MERGE и какие в нем подводные камни?**

### **Правильный ответ**
> "MERGE — это операция "upsert" (обновить или вставить), которая позволяет в одной инструкции выполнять INSERT, UPDATE, DELETE на основе сравнения с исходной таблицей. Главные подводные камни: необходимость завершать точкой с запятой, риск ошибок при неправильном условии WHEN, и блокировки на всю таблицу."

### **Пример MERGE**
```sql
MERGE TargetTable AS target
USING SourceTable AS source
    ON target.ID = source.ID
WHEN MATCHED THEN
    UPDATE SET target.Value = source.Value
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ID, Value) VALUES (source.ID, source.Value)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE
OUTPUT $action, Inserted.*, Deleted.*;
```

### **Опасные подводные камни**

1. **Обязательная точка с запятой**
   ```sql
   -- Ошибка!
   BEGIN TRANSACTION
   MERGE ... -- Должна быть ; перед следующим оператором
   COMMIT
   
   -- Правильно
   BEGIN TRANSACTION;
   MERGE ...;
   COMMIT;
   ```

2. **Несколько WHEN MATCHED**
   ```sql
   MERGE ...
   WHEN MATCHED AND source.Status = 'Active' THEN
       UPDATE ...
   WHEN MATCHED AND source.Status = 'Inactive' THEN
       DELETE  -- ОШИБКА! Нельзя UPDATE и DELETE в одном MATCHED
   ```

3. **Рекурсивные триггеры**
   - MERGE может вызвать триггеры несколько раз для одной строки

4. **Блокировки**
   - MERSE часто блокирует всю таблицу, даже если изменяет 1 строку

### **Альтернатива MERGE**
```sql
-- Более предсказуемый вариант
BEGIN TRANSACTION

UPDATE target 
SET Value = source.Value
FROM TargetTable target
INNER JOIN SourceTable source ON target.ID = source.ID

INSERT INTO TargetTable (ID, Value)
SELECT source.ID, source.Value
FROM SourceTable source
LEFT JOIN TargetTable target ON source.ID = target.ID
WHERE target.ID IS NULL

COMMIT
```

---
