## **Вопрос 10: В чем разница между CHAR, VARCHAR, NVARCHAR? Когда что использовать?**

Этот вопрос проверяет понимание хранения данных и кодировок.

### **Короткий ответ для собеседования**
> "CHAR — фиксированной длины, дополняется пробелами, тратит память, но быстрее для одинаковых длин. VARCHAR — переменной длины, экономит место, но накладные расходы на хранение длины. NVARCHAR — для Unicode (2 байта на символ), обязателен для международных данных. CHAR для кодов (MFO, INN), VARCHAR для имен/описаний, NVARCHAR когда нужны любые языки."

### **Детальное сравнение**

```sql
-- Создаем тестовую таблицу
CREATE TABLE CharTest (
    ID INT IDENTITY,
    Code CHAR(10),        -- Всегда 10 байт (латиница) / 10 байт (кириллица в русской кодировке)
    Name VARCHAR(50),     -- Фактическая длина + 2 байта
    Description NVARCHAR(100) -- Фактическая длина × 2 + 2 байта
)

INSERT INTO CharTest VALUES 
('ABC', 'Иван', N'Иван'),        -- Латинский код
('0123456789', 'Petrov', N'Петров') -- Полный CHAR
```

### **Хранение в памяти**

```sql
-- Что на самом деле хранится
/*
Данные: Code='ABC', Name='Иван', Description=N'Иван'

CHAR(10):    'ABC       '  (10 байт: 3 буквы + 7 пробелов)
VARCHAR(50): 'Иван'        (4 + 2 = 6 байт в русской кодировке)
NVARCHAR(100): N'Иван'     (4×2 + 2 = 10 байт)

Данные: Code='0123456789', Name='Petrov', Description=N'Петров'

CHAR(10):    '0123456789'  (10 байт)
VARCHAR(50): 'Petrov'      (6 + 2 = 8 байт)  
NVARCHAR(100): N'Петров'   (6×2 + 2 = 14 байт)
*/
```

### **Производительность: тест JOIN**

```sql
-- Тестовая таблица с 100k строк
CREATE TABLE TestJoin (
    ID INT PRIMARY KEY,
    CharKey CHAR(10),      -- Фиксированная длина
    VarKey VARCHAR(10),    -- Переменная длина
    NVarKey NVARCHAR(10)   -- Unicode
)

-- Наполняем данными
INSERT INTO TestJoin (ID, CharKey, VarKey, NVarKey)
SELECT 
    n,
    LEFT(NEWID(), 10),
    LEFT(NEWID(), RAND()*9+1),  -- Разная длина
    LEFT(NEWID(), RAND()*9+1)
FROM (
    SELECT TOP 100000 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
) t

-- Создаем индексы
CREATE INDEX IX_Char ON TestJoin(CharKey)
CREATE INDEX IX_Var ON TestJoin(VarKey)  
CREATE INDEX IX_NVar ON TestJoin(NVarKey)

-- Тест производительности
SET STATISTICS IO, TIME ON

-- JOIN по CHAR (быстрее, предсказуемый размер)
SELECT COUNT(*) 
FROM TestJoin t1
INNER JOIN TestJoin t2 ON t1.CharKey = t2.CharKey

-- JOIN по VARCHAR (медленнее, надо сравнивать длины)
SELECT COUNT(*)
FROM TestJoin t1  
INNER JOIN TestJoin t2 ON t1.VarKey = t2.VarKey

-- JOIN по NVARCHAR (еще медленнее, в 2 раза больше данных)
SELECT COUNT(*)
FROM TestJoin t1
INNER JOIN TestJoin t2 ON t1.NVarKey = t2.NVarKey
```

**Результаты (примерные):**
- CHAR JOIN: ~200 ms, 500 logical reads
- VARCHAR JOIN: ~300 ms, 600 logical reads  
- NVARCHAR JOIN: ~500 ms, 900 logical reads

### **Проблемы сравнения и сортировки**

```sql
-- 1. Сравнение CHAR с VARCHAR
DECLARE @c CHAR(5) = 'ABC'   -- 'ABC  '
DECLARE @v VARCHAR(5) = 'ABC' -- 'ABC'

SELECT 
    CASE WHEN @c = @v THEN 'Equal' ELSE 'Not equal' END AS DirectCompare,
    -- Result: 'Equal' (SQL Server обрезает пробелы при сравнении)
    
    CASE WHEN @c LIKE @v THEN 'LIKE equal' ELSE 'Not equal' END AS LikeCompare
    -- Result: 'LIKE equal'

-- 2. Проблемы с LIKE
SELECT * FROM Table WHERE CharColumn LIKE 'ABC%'   -- Использует индекс
SELECT * FROM Table WHERE CharColumn LIKE 'ABC  %' -- Не использует индекс!

-- 3. Сортировка (Collation matters!)
SELECT * FROM Table ORDER BY CharColumn  -- Быстро (фиксированная длина)
SELECT * FROM Table ORDER BY VarColumn   -- Медленнее
```

### **Каверзные уточняющие вопросы**

1. **"Что лучше для индекса: CHAR(10) или VARCHAR(10)?"**
   - **Ответ**: Если данные всегда заполняют поле (например, код из 10 символов) — CHAR. Если длина разная — VARCHAR. CHAR даст более предсказуемую производительность.

2. **"Почему NVARCHAR в 2 раза больше, если у меня только английские буквы?"**
   - **Ответ**: SQL Server резервирует 2 байта на любой символ в NVARCHAR, даже для латиницы. Для только английских текстов используйте VARCHAR.

3. **"Что такое VARCHAR(MAX) и когда использовать?"**
   - **Ответ**: Хранит до 2ГБ, данные могут храниться вне строки (LOB). Использовать когда:
     - Текст может превысить 8000 байт
     - Не нужно индексировать столбец
     - Редкие SELECT этого столбца
   ```sql
   -- Плохо для частых запросов
   CREATE TABLE Articles (
       ID INT,
       Content VARCHAR(MAX)  -- Медленный доступ если > 8000 байт
   )
   
   -- Лучше разделить
   CREATE TABLE Articles (
       ID INT,
       Summary VARCHAR(500),  -- Для списков
       Content VARCHAR(MAX)   -- Для полного текста
   )
   ```

4. **"Как выбрать длину для VARCHAR?"**
   - **Ответ**: На основе бизнес-правил, но с запасом:
   ```sql
   -- Плохо: постоянно менять длину
   FirstName VARCHAR(20) -- А если двойное имя?
   
   -- Лучше: анализ данных + запас
   SELECT 
       MAX(LEN(FirstName)) AS MaxLength,
       AVG(LEN(FirstName)) AS AvgLength
   FROM ImportedData
   
   -- Хорошая практика: 
   FirstName VARCHAR(50)  -- С запасом
   ```

### **Практические рекомендации**

```sql
-- 1. Коды и идентификаторы фиксированной длины
CREATE TABLE Banks (
    MFO CHAR(6) PRIMARY KEY,        -- Всегда 6 цифр
    Name NVARCHAR(100)              -- Может быть на любом языке
)

-- 2. Персональные данные (международные)
CREATE TABLE Users (
    Email VARCHAR(255),             -- ASCII достаточно
    FullName NVARCHAR(100),         -- Unicode для любых имен
    Phone VARCHAR(20)               -- Цифры + символы
)

-- 3. Системные коды
CREATE TABLE Statuses (
    Code CHAR(3) PRIMARY KEY,       -- 'ACT', 'DEL', 'PND'
    Description NVARCHAR(50)
)

-- 4. Адреса
CREATE TABLE Addresses (
    ZipCode CHAR(10),               -- Фиксированный формат
    City NVARCHAR(50),              Unicode
    Street NVARCHAR(100)
)
```

---

## **Вопрос 11: В чем разница между TRUNCATE и DELETE?**

Этот вопрос проверяет понимание операций DML и их последствий.

### **Короткий ответ для собеседования**
> "DELETE — операция DML, логируется каждая строка, можно использовать WHERE, срабатывают триггеры, медленно для больших таблиц. TRUNCATE — операция DDL, логируется только освобождение страниц, нельзя использовать WHERE, не срабатывают триггеры, очень быстро, но требует больше прав."

### **Детальное сравнение**

```sql
-- Создаем тестовую таблицу
CREATE TABLE LogTest (
    ID INT IDENTITY PRIMARY KEY,
    Data VARCHAR(100),
    Modified DATETIME DEFAULT GETDATE()
)

CREATE TRIGGER trg_LogTest_Delete 
ON LogTest AFTER DELETE
AS
BEGIN
    INSERT INTO DeleteLog SELECT *, GETDATE() FROM deleted
END

-- Наполняем данными (1 млн строк)
INSERT INTO LogTest (Data)
SELECT TOP 1000000 'Data_' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS VARCHAR)
FROM sys.all_objects a CROSS JOIN sys.all_objects b CROSS JOIN sys.all_objects c
```

### **Сравнение в таблице**

| Критерий | DELETE | TRUNCATE |
|----------|--------|----------|
| **Тип операции** | DML | DDL |
| **Логирование** | Каждая строка | Только страницы |
| **WHERE** | Можно | Нельзя |
| **Триггеры** | Срабатывают | Не срабатывают |
| **Скорость** | Медленно | Очень быстро |
| **Права** | DELETE на таблицу | ALTER на таблицу |
| **IDENTITY** | Не сбрасывает | Сбрасывает (если нет FOREIGN KEY) |
| **Locking** | Row/Page locks | Schema lock на таблицу |
| **Ограничения** | Проверяет FK | Не работает при FK ссылках |
| **Восстановление** | Можно откатить | Можно откатить (но в транзакции!) |

### **Производительность: наглядный тест**

```sql
-- Копируем таблицу для теста
SELECT * INTO TestDelete FROM LogTest  -- 1 млн строк
SELECT * INTO TestTruncate FROM LogTest

-- Тест DELETE
SET STATISTICS TIME, IO ON

BEGIN TRANSACTION
DELETE FROM TestDelete  -- Минуты!
-- В логе: 1 млн записей
COMMIT
-- Время: ~2 минуты, Log Size: +500 MB

-- Тест TRUNCATE
BEGIN TRANSACTION  
TRUNCATE TABLE TestTruncate  -- Секунды!
-- В логе: ~10 записей (освобождение экстентов)
COMMIT
-- Время: ~1 секунда, Log Size: +0.1 MB
```

### **Ограничения TRUNCATE**

```sql
-- 1. FOREIGN KEY constraint
CREATE TABLE Parent (ID INT PRIMARY KEY)
CREATE TABLE Child (ID INT PRIMARY KEY, ParentID INT REFERENCES Parent(ID))

INSERT INTO Parent VALUES (1), (2)
INSERT INTO Child VALUES (1, 1), (2, 2)

TRUNCATE TABLE Parent  -- ОШИБКА: Cannot truncate table 'Parent' because it is being referenced by a FOREIGN KEY constraint.

-- Решение: удалить constraint или сначала Child
DELETE FROM Child
TRUNCATE TABLE Parent

-- 2. Участвует в индексированном представлении
CREATE VIEW VW_Test WITH SCHEMABINDING
AS
SELECT ID, COUNT_BIG(*) AS Cnt
FROM dbo.TestTruncate
GROUP BY ID

CREATE UNIQUE CLUSTERED INDEX IX_VW ON VW_Test(ID)

TRUNCATE TABLE TestTruncate  -- ОШИБКА: Cannot truncate table 'TestTruncate' because it is being referenced by object 'VW_Test'.

-- 3. Транзакционная репликация
-- Если таблица опубликована - TRUNCATE не сработает
```

### **Каверзные уточняющие вопросы**

1. **"Можно ли откатить TRUNCATE?"**
   - **Ответ**: Да, если он выполнен в явной транзакции:
   ```sql
   BEGIN TRANSACTION
   TRUNCATE TABLE BigTable
   SELECT COUNT(*) FROM BigTable  -- 0
   ROLLBACK
   SELECT COUNT(*) FROM BigTable  -- Исходное количество
   ```

2. **"TRUNCATE сбрасывает IDENTITY. Как быть если нужно сохранить?"**
   - **Ответ**: Использовать DELETE без WHERE или сохранить seed:
   ```sql
   -- Способ 1: Узнать текущее значение
   DECLARE @nextID INT = IDENT_CURRENT('TableName')
   TRUNCATE TABLE TableName
   DBCC CHECKIDENT ('TableName', RESEED, @nextID)
   
   -- Способ 2: DELETE в batches
   WHILE 1=1
   BEGIN
       DELETE TOP (10000) FROM BigTable
       IF @@ROWCOUNT = 0 BREAK
   END
   ```

3. **"Как удалить все строки из таблицы с FK constraint?"**
   - **Ответ**: Либо временно отключить constraint, либо удалять в правильном порядке:
   ```sql
   -- Способ 1: Отключить FK
   ALTER TABLE Child NOCHECK CONSTRAINT ALL
   TRUNCATE TABLE Parent
   TRUNCATE TABLE Child
   ALTER TABLE Child CHECK CONSTRAINT ALL
   
   -- Способ 2: DELETE в порядке зависимостей
   DELETE FROM Child
   DELETE FROM Parent  -- или TRUNCATE если нет других ограничений
   ```

4. **"Что быстрее: DELETE без WHERE или TRUNCATE?"**
   - **Ответ**: TRUNCATE в 100-1000 раз быстрее для больших таблиц. Но если таблица маленькая (< 1000 строк) - разница незначительна.

### **Интересные особенности**

```sql
-- 1. TRUNCATE можно использовать в блоке TRY-CATCH
BEGIN TRY
    BEGIN TRANSACTION
    TRUNCATE TABLE ImportantTable
    COMMIT
END TRY
BEGIN CATCH
    ROLLBACK
    -- Обработка ошибки
END CATCH

-- 2. TRUNCATE освобождает страницы (можно увидеть в sys.allocation_units)
SELECT 
    OBJECT_NAME(p.object_id) AS TableName,
    au.total_pages,
    au.used_pages
FROM sys.allocation_units au
INNER JOIN sys.partitions p ON au.container_id = p.partition_id
WHERE OBJECT_NAME(p.object_id) = 'BigTable'

-- До TRUNCATE: total_pages = 10000
-- После TRUNCATE: total_pages = 0 (страницы возвращены в пул)

-- 3. TRUNCATE не работает на системных таблицах
TRUNCATE TABLE sys.objects  -- ОШИБКА!

-- 4. TRUNCATE можно использовать с PARTITION
CREATE PARTITION FUNCTION pfTest (INT) AS RANGE FOR VALUES (100, 200, 300)
CREATE PARTITION SCHEME psTest AS PARTITION pfTest ALL TO ([PRIMARY])

CREATE TABLE PartitionedTable (ID INT, Data VARCHAR(100)) ON psTest(ID)
INSERT INTO PartitionedTable SELECT 1, 'A' UNION SELECT 150, 'B'

-- Очистить только одну партицию
TRUNCATE TABLE PartitionedTable WITH (PARTITIONS (2))  -- Только partition 2
```

### **Безопасность и разрешения**

```sql
-- Кто может выполнять TRUNCATE?
-- 1. Владелец таблицы (dbo)
-- 2. Члены ролей: db_ddladmin, db_owner
-- 3. Пользователи с разрешением ALTER на таблицу

-- Дать разрешение
GRANT ALTER ON TableName TO UserName

-- Отобрать разрешение
REVOKE ALTER ON TableName FROM UserName

-- Проверить разрешения
EXECUTE AS USER = 'UserName'
SELECT HAS_PERMS_BY_NAME('dbo.TableName', 'OBJECT', 'ALTER')
REVERT
```

### **Практическое правило**
> "Используйте DELETE когда:
> - Нужно логировать каждое удаление
> - Должны сработать триггеры  
> - Нужно удалить часть строк (WHERE)
> - Таблица участвует в репликации
>
> Используйте TRUNCATE когда:
> - Нужно быстро удалить ВСЕ строки
> - Таблица большая (> 10000 строк)
> - Не нужны триггеры
> - Можно получить ALTER права
> - Нет FOREIGN KEY ограничений"

---
