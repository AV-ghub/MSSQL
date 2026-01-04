## **Что такое ACID? Систематизация**

**ACID** — это набор свойств, гарантирующих надежность транзакций в СУБД. В SQL Server эти свойства обеспечиваются комплексно на разных уровнях.

### **A — Atomicity (Атомарность)**
> "Транзакция выполняется полностью или не выполняется совсем."

**Как обеспечивается в SQL Server:**
```sql
-- Явное управление транзакцией
BEGIN TRANSACTION
BEGIN TRY
    UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1
    UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2
    -- Обе операции должны выполниться
    COMMIT TRANSACTION  -- Фиксация
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION  -- Полный откат
    THROW
END CATCH
```

**Механизмы:**
- **Transaction Log (журнал транзакций)** — записывает все изменения
- **Write-Ahead Logging (WAL)** — сначала лог, потом данные
- **ROLLBACK command** — откат по команде или при ошибке

**Связь с ранее изученным:**
- Свойство `TRUNCATE` — операция DDL, но тоже атомарна
- `@@TRANCOUNT` — отслеживание уровня вложенности транзакций

### **C — Consistency (Согласованность)**
> "Транзакция переводит базу из одного согласованного состояния в другое."

**Пример нарушения:**
```sql
-- Нарушение foreign key constraint
BEGIN TRANSACTION
INSERT INTO Orders (OrderID, CustomerID) VALUES (1001, 9999)
-- CustomerID = 9999 не существует в Customers
COMMIT  -- ОШИБКА! Транзакция откатится
```

**Как обеспечивается:**
1. **Constraints (ограничения)**
   ```sql
   PRIMARY KEY, FOREIGN KEY, CHECK, UNIQUE, NOT NULL
   ```
2. **Triggers (триггеры)**
   ```sql
   CREATE TRIGGER trg_Consistency ON Orders AFTER INSERT
   AS
   BEGIN
       IF EXISTS (SELECT 1 FROM inserted i 
                  LEFT JOIN Customers c ON i.CustomerID = c.CustomerID
                  WHERE c.CustomerID IS NULL)
       BEGIN
           ROLLBACK TRANSACTION
           RAISERROR('Invalid CustomerID', 16, 1)
       END
   END
   ```
3. **Stored Procedures (хранимые процедуры)** — инкапсуляция бизнес-правил

### **I — Isolation (Изолированность)**
> "Транзакции выполняются изолированно друг от друга."

**Уровни изоляции в SQL Server (от слабого к сильному):**
```sql
-- Уровни изоляции
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- "грязное чтение"
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;    -- по умолчанию
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- защита от неповторяемого чтения
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;          -- изоляция через snapshot
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;      -- полная сериализация
```

**Проблемы, которые решают уровни изоляции:**
1. **Dirty Read (чтение "грязных" данных)** — `READ UNCOMMITTED`
2. **Non-repeatable Read (неповторяемое чтение)** — `REPEATABLE READ`
3. **Phantom Read (чтение "фантомов")** — `SERIALIZABLE`
4. **Lost Update (потерянное обновление)** — блокировки

**Связь с deadlock:**
```sql
-- Изоляция влияет на блокировки
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
BEGIN TRANSACTION
SELECT * FROM Accounts  -- Блокировки диапазонов
-- Другая транзакция не может вставить в этот диапазон
-- Может привести к deadlock при конкурентном доступе
```

### **D — Durability (Долговечность)**
> "После фиксации транзакции изменения сохраняются даже при сбое системы."

**Механизмы обеспечения:**
1. **Write-Ahead Logging (WAL)**
   ```sql
   -- Изменение: 1) запись в журнал, 2) запись в данные
   -- При сбое: восстановление из журнала (REDO/UNDO)
   ```
2. **Checkpoint (контрольная точка)**
   ```sql
   -- Периодическая запись "грязных" страниц на диск
   CHECKPOINT  -- Принудительная контрольная точка
   ```
3. **Backup/Restore (резервное копирование)**
   ```sql
   -- Полная гарантия долговечности
   BACKUP DATABASE MyDB TO DISK = 'C:\Backup\MyDB.bak'
   ```

## **Визуализация ACID в SQL Server**

```
[Транзакция начинается]
    │
    ├─ Atomicity ──┐
    │              │
    │   BEGIN TRAN │
    │      │       │
    │   Операция 1─┤──► Запись в Transaction Log
    │      │       │
    │   Операция 2─┤──► Запись в Transaction Log  
    │      │       │
    │   Ошибка? ───┤─── Да ───► ROLLBACK (откат из лога)
    │      │       │     │
    │      Нет     │     │
    │      │       │     │
    │   COMMIT ────┤◄────┘
    │              │   COMMIT ───► Фиксация в логе
    │              │              (готово для восстановления)
    │
    ├─ Consistency─┤
    │   │          │
    │   Проверка ──┤──► Constraints, Triggers
    │   ограничений│
    │              │
    ├─ Isolation ──┤
    │   │          │
    │   Блокировки─┤──► Locks (Row, Page, Table)
    │   │          │    Изоляция через SNAPSHOT
    │   Видимость ─┤──► Read View (для SNAPSHOT)
    │              │
    └─ Durability ─┤
                   │
            Checkpoint ───► Сброс на диск
                   │
            Backup ───────► Резервная копия
```

## **Практические примеры ACID**

### **Пример 1: Банковский перевод (полный ACID)**
```sql
CREATE PROCEDURE TransferMoney
    @FromAccount INT,
    @ToAccount INT, 
    @Amount DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;  -- Автоматический ROLLBACK при ошибке
    
    BEGIN TRANSACTION  -- Atomicity: начало транзакции
    
    -- Isolation: блокировки строк
    UPDATE Accounts WITH (ROWLOCK, UPDLOCK) 
    SET Balance = Balance - @Amount
    WHERE AccountID = @FromAccount AND Balance >= @Amount
    
    IF @@ROWCOUNT = 0
    BEGIN
        ROLLBACK TRANSACTION
        RAISERROR('Insufficient funds', 16, 1)
        RETURN
    END
    
    -- Consistency: проверка существования счета
    IF NOT EXISTS (SELECT 1 FROM Accounts WHERE AccountID = @ToAccount)
    BEGIN
        ROLLBACK TRANSACTION
        RAISERROR('Target account not found', 16, 1)
        RETURN
    END
    
    UPDATE Accounts WITH (ROWLOCK)
    SET Balance = Balance + @Amount
    WHERE AccountID = @ToAccount
    
    -- Логирование (часть согласованности)
    INSERT INTO Transactions (FromAccount, ToAccount, Amount, Timestamp)
    VALUES (@FromAccount, @ToAccount, @Amount, GETDATE())
    
    COMMIT TRANSACTION  -- Durability: фиксация
    
    -- Долговечность: синхронная запись
    CHECKPOINT  -- Принудительная запись на диск
END
```

### **Пример 2: Нарушение ACID и последствия**

```sql
-- НАРУШЕНИЕ Atomicity: нет явной транзакции
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1
-- Сбой сервера здесь...
UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2
-- Результат: 100 списано, но не зачислено!

-- НАРУШЕНИЕ Consistency: отключенные constraints
ALTER TABLE Orders NOCHECK CONSTRAINT FK_Orders_Customers
INSERT INTO Orders (CustomerID) VALUES (9999)  -- Несуществующий клиент
ALTER TABLE Orders CHECK CONSTRAINT FK_Orders_CustomERS
-- База в несогласованном состоянии!

-- НАРУШЕНИЕ Isolation: READ UNCOMMITTED
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED
SELECT * FROM Accounts  -- Видит незакоммиченные данные

-- НАРУШЕНИЕ Durability: задержка записи
ALTER DATABASE MyDB SET DELAYED_DURABILITY = FORCED
COMMIT TRANSACTION  -- Фиксация отложена, риск потери данных
```

## **Каверзные вопросы по ACID на собеседовании**

1. **"Можно ли отключить ACID свойства?"**
   - **Ответ**: Частично. Например:
     - `SET IMPLICIT_TRANSACTIONS` — изменяет атомарность
     - `ALTER DATABASE ... SET DELAYED_DURABILITY` — ослабляет долговечность
     - `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` — ослабляет изоляцию
     - `NOCHECK CONSTRAINT` — ослабляет согласованность

2. **"Как ACID связан с Recovery Model?"**
   - **Ответ**: Recovery Model влияет на Durability:
   ```sql
   -- FULL: полное восстановление (ACID-совместимо)
   -- BULK_LOGGED: ограниченное восстановление массовых операций
   -- SIMPLE: точка восстановления только до последнего backup
   ```

3. **"Что такое XACT_ABORT и как он связан с ACID?"**
   - **Ответ**: `SET XACT_ABORT ON` обеспечивает атомарность при ошибках:
   ```sql
   SET XACT_ABORT ON
   BEGIN TRANSACTION
   INSERT INTO Table1 VALUES (1)  -- Ошибка
   INSERT INTO Table2 VALUES (2)  -- Не выполнится, автоматический ROLLBACK
   COMMIT
   ```

4. **"Как ACID реализован в памяти-оптимизированных таблицах?"**
   - **Ответ**: In-Memory OLTP использует MVCC (Multiversion Concurrency Control) для изоляции и оптимистичные блокировки.

## **ACID и темы, которые мы уже разобрали**

| Тема из наших обсуждений | Связь с ACID |
|-------------------------|--------------|
| **Deadlock** | Проблема Isolation. Решение через порядок блокировок (Consistency) |
| **Транзакции (BEGIN/COMMIT/ROLLBACK)** | Механизм Atomicity |
| **Уровни изоляции** | Реализация Isolation |
| **FOREIGN KEY, CHECK constraints** | Механизм Consistency |
| **Transaction Log** | Основа Durability + Atomicity |
| **BACKUP/RESTORE** | Гарантия Durability |
| **TRUNCATE vs DELETE** | Обе операции атомарны, но разная изоляция |
| **NOLOCK hint** | Нарушение Isolation (READ UNCOMMITTED) |
| **Snapshot Isolation** | Альтернативный механизм Isolation |

---
