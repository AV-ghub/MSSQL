## 🔍 **Часть 1: Создание сеанса Extended Events для захвата Deadlock'ов**

SQL Server уже ведет запись deadlock'ов в стандартный системный сеанс **`system_health`**. Но для удобства и фильтрации лучше создать свой сеанс.

### **Вариант 1: Через T-SQL (рекомендуется)**
```sql
-- Создание сеанса для захвата deadlock'ов
CREATE EVENT SESSION [DeadlockMonitor] ON SERVER 
ADD EVENT sqlserver.xml_deadlock_report(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_name,
        sqlserver.sql_text,
        sqlserver.username
    )
) 
ADD TARGET package0.event_file(
    SET filename = N'C:\XELogs\DeadlockMonitor.xel', -- Укажите свой путь
    max_file_size = 100, -- MB
    max_rollover_files = 5
)
WITH (
    MAX_MEMORY = 4 MB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    STARTUP_STATE = ON -- Сеанс запустится автоматически при старте SQL Server
);
GO

-- Запуск сеанса
ALTER EVENT SESSION [DeadlockMonitor] ON SERVER STATE = START;
```

### **Вариант 2: Через GUI в SSMS**
1. **Management → Extended Events → Sessions → New Session**
2. На вкладке **Events**: 
   - Найти и добавить `xml_deadlock_report`
   - Нажать **Configure → Global Fields (Actions)** и добавить:
     - `client_app_name`
     - `client_hostname` 
     - `database_name`
     - `sql_text`
     - `username`
3. На вкладке **Data Storage**: Выбрать `event_file`, указать путь
4. На вкладке **Advanced**: Установить `Dispatch Latency = 30 seconds`
5. **ОК → Start the session immediately**

## 📊 **Часть 2: Парсинг .xel-файлов в удобную таблицу**

### **Базовый парсинг для быстрого анализа**
```sql
-- Чтение данных из файлов сеанса
SELECT 
    DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
           event_data.value('(@timestamp)[1]', 'datetime2')) AS EventTime,
    event_data.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(255)') AS Application,
    event_data.value('(action[@name="client_hostname"]/value)[1]', 'nvarchar(255)') AS HostName,
    event_data.value('(action[@name="database_name"]/value)[1]', 'nvarchar(255)') AS DatabaseName,
    event_data.value('(action[@name="username"]/value)[1]', 'nvarchar(255)') AS UserName,
    event_data.value('(action[@name="sql_text"]/value)[1]', 'nvarchar(max)') AS SqlText,
    CAST(event_data AS XML) AS DeadlockGraph -- Полный XML deadlock graph
FROM sys.fn_xe_file_target_read_file(
    'C:\XELogs\DeadlockMonitor*.xel', -- * для чтения всех файлов
    NULL, NULL, NULL
)
CROSS APPLY (SELECT CAST(event_data AS XML) AS event_data) AS ed
WHERE event_data.value('(@name)[1]', 'nvarchar(255)') = 'xml_deadlock_report'
ORDER BY EventTime DESC;
```

### **Расширенный парсинг: извлечение деталей Deadlock Graph**
Этот запрос парсит сам XML deadlock'а, извлекая ключевую информацию о жертвах и участниках.

```sql
WITH DeadlockData AS (
    SELECT 
        DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
               event_data.value('(@timestamp)[1]', 'datetime2')) AS EventTime,
        event_data.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(255)') AS Application,
        event_data.value('(action[@name="database_name"]/value)[1]', 'nvarchar(255)') AS DatabaseName,
        CAST(event_data AS XML) AS DeadlockGraph
    FROM sys.fn_xe_file_target_read_file(
        'C:\XELogs\DeadlockMonitor*.xel',
        NULL, NULL, NULL
    )
    CROSS APPLY (SELECT CAST(event_data AS XML) AS event_data) AS ed
    WHERE event_data.value('(@name)[1]', 'nvarchar(255)') = 'xml_deadlock_report'
)
SELECT 
    dd.EventTime,
    dd.Application,
    dd.DatabaseName,
    -- Жертва deadlock'а
    dd.DeadlockGraph.value('(//deadlock/victim-list/victimProcess/@id)[1]', 'nvarchar(50)') AS VictimProcessId,
    
    -- Участник 1
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/@id', 'nvarchar(50)') AS Process1_Id,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/@lockMode', 'nvarchar(10)') AS Process1_LockMode,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/@clientapp', 'nvarchar(255)') AS Process1_App,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/@isolationlevel', 'nvarchar(50)') AS Process1_IsolationLevel,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/executionStack/frame[1]/@procname[1]', 'nvarchar(255)') AS Process1_ProcName,
    LEFT(dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[1]])[1]/executionStack/frame[1]/@sqlhandle[1]', 'nvarchar(255)'), 50) AS Process1_SqlHandle,
    
    -- Участник 2  
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[2]])[1]/@id', 'nvarchar(50)') AS Process2_Id,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[2]])[1]/@lockMode', 'nvarchar(10)') AS Process2_LockMode,
    dd.DeadlockGraph.value('(//deadlock/process-list/process[@id[2]])[1]/@clientapp', 'nvarchar(255)') AS Process2_App,
    
    -- Ресурс из-за которого конфликт
    dd.DeadlockGraph.value('(//deadlock/resource-list/*[1]/@objectname)[1]', 'nvarchar(255)') AS ContendedObject,
    dd.DeadlockGraph.value('(//deadlock/resource-list/*[1]/@dbid)[1]', 'int') AS DatabaseId,
    
    -- Полный граф для детального анализа
    dd.DeadlockGraph
FROM DeadlockData dd
ORDER BY dd.EventTime DESC;
```

## 🛠 **Часть 3: Готовое решение для мониторинга Deadlock'ов**

### **Шаг 1: Создание таблицы для хранения истории**
```sql
CREATE TABLE dbo.DeadlockHistory (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    EventTime DATETIME2 NOT NULL,
    Application NVARCHAR(255),
    HostName NVARCHAR(255),
    DatabaseName NVARCHAR(255),
    UserName NVARCHAR(255),
    VictimProcessId NVARCHAR(50),
    Process1_ProcName NVARCHAR(255),
    Process1_SqlHandle NVARCHAR(255),
    Process2_App NVARCHAR(255),
    ContendedObject NVARCHAR(255),
    DeadlockGraph XML,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    INDEX IX_EventTime (EventTime),
    INDEX IX_Database (DatabaseName)
);
```

### **Шаг 2: Процедура для автоматического парсинга и сохранения**
```sql
CREATE PROCEDURE dbo.ParseAndSaveDeadlocks
    @FilePath NVARCHAR(500) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Если путь не указан, используем путь по умолчанию
    IF @FilePath IS NULL
        SET @FilePath = 'C:\XELogs\DeadlockMonitor*.xel';
    
    -- Временная таблица для новых deadlock'ов
    DECLARE @NewDeadlocks TABLE (
        EventTime DATETIME2,
        Application NVARCHAR(255),
        HostName NVARCHAR(255),
        DatabaseName NVARCHAR(255),
        UserName NVARCHAR(255),
        VictimProcessId NVARCHAR(50),
        Process1_ProcName NVARCHAR(255),
        Process1_SqlHandle NVARCHAR(255),
        Process2_App NVARCHAR(255),
        ContendedObject NVARCHAR(255),
        DeadlockGraph XML
    );
    
    -- Парсинг файлов
    INSERT INTO @NewDeadlocks
    SELECT 
        DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
               event_data.value('(@timestamp)[1]', 'datetime2')) AS EventTime,
        event_data.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(255)') AS Application,
        event_data.value('(action[@name="client_hostname"]/value)[1]', 'nvarchar(255)') AS HostName,
        event_data.value('(action[@name="database_name"]/value)[1]', 'nvarchar(255)') AS DatabaseName,
        event_data.value('(action[@name="username"]/value)[1]', 'nvarchar(255)') AS UserName,
        CAST(event_data AS XML).value('(//deadlock/victim-list/victimProcess/@id)[1]', 'nvarchar(50)') AS VictimProcessId,
        CAST(event_data AS XML).value('(//deadlock/process-list/process[@id[1]])[1]/executionStack/frame[1]/@procname[1]', 'nvarchar(255)') AS Process1_ProcName,
        LEFT(CAST(event_data AS XML).value('(//deadlock/process-list/process[@id[1]])[1]/executionStack/frame[1]/@sqlhandle[1]', 'nvarchar(255)'), 50) AS Process1_SqlHandle,
        CAST(event_data AS XML).value('(//deadlock/process-list/process[@id[2]])[1]/@clientapp', 'nvarchar(255)') AS Process2_App,
        CAST(event_data AS XML).value('(//deadlock/resource-list/*[1]/@objectname)[1]', 'nvarchar(255)') AS ContendedObject,
        CAST(event_data AS XML) AS DeadlockGraph
    FROM sys.fn_xe_file_target_read_file(@FilePath, NULL, NULL, NULL)
    CROSS APPLY (SELECT CAST(event_data AS XML) AS event_data) AS ed
    WHERE event_data.value('(@name)[1]', 'nvarchar(255)') = 'xml_deadlock_report'
        -- Исключаем уже сохраненные deadlock'ы
        AND DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
                   event_data.value('(@timestamp)[1]', 'datetime2')) > 
                   ISNULL((SELECT MAX(EventTime) FROM dbo.DeadlockHistory), '1900-01-01');
    
    -- Сохранение в основную таблицу
    INSERT INTO dbo.DeadlockHistory (
        EventTime, Application, HostName, DatabaseName, UserName,
        VictimProcessId, Process1_ProcName, Process1_SqlHandle,
        Process2_App, ContendedObject, DeadlockGraph
    )
    SELECT * FROM @NewDeadlocks;
    
    -- Возвращаем статистику
    SELECT 
        @@ROWCOUNT AS DeadlocksSaved,
        COUNT(*) AS TotalInHistory
    FROM dbo.DeadlockHistory;
END;
GO
```

### **Шаг 3: Задание для автоматического запуска (SQL Agent Job)**
```sql
-- Создание задания для ежедневного парсинга
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Parse Deadlock Logs',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = N'Parse Deadlock Logs',
    @step_name = N'Parse and Save Deadlocks',
    @subsystem = N'TSQL',
    @command = N'EXEC YourDatabase.dbo.ParseAndSaveDeadlocks;',
    @database_name = N'YourDatabase';

EXEC sp_add_schedule
    @schedule_name = N'Daily Midnight',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 10000; -- 00:00:00

EXEC sp_attach_schedule
    @job_name = N'Parse Deadlock Logs',
    @schedule_name = N'Daily Midnight';

EXEC sp_add_jobserver
    @job_name = N'Parse Deadlock Logs';
```

## 🔎 **Часть 4: Анализ собранных данных**

### **Частые deadlock'ы по базам/приложениям**
```sql
SELECT 
    DatabaseName,
    Application,
    COUNT(*) AS DeadlockCount,
    MIN(EventTime) AS FirstOccurrence,
    MAX(EventTime) AS LastOccurrence
FROM dbo.DeadlockHistory
GROUP BY DatabaseName, Application
ORDER BY DeadlockCount DESC;
```

### **Поиск проблемных объектов**
```sql
SELECT 
    ContendedObject,
    DatabaseName,
    COUNT(*) AS ConflictCount,
    STRING_AGG(DISTINCT Application, ', ') AS ApplicationsInvolved
FROM dbo.DeadlockHistory
WHERE ContendedObject IS NOT NULL
GROUP BY ContendedObject, DatabaseName
ORDER BY ConflictCount DESC;
```

### **Анализ deadlock graph через XQuery**
```sql
-- Детальный анализ конкретного deadlock'а
DECLARE @DeadlockId INT = 1; -- ID из таблицы DeadlockHistory

SELECT
    -- Извлекаем все процессы из графа
    ProcessData.Process.value('@id', 'NVARCHAR(50)') AS ProcessId,
    ProcessData.Process.value('@lockMode', 'NVARCHAR(10)') AS LockMode,
    ProcessData.Process.value('@clientapp', 'NVARCHAR(255)') AS ClientApp,
    ProcessData.Process.value('@isolationlevel', 'NVARCHAR(50)') AS IsolationLevel,
    ProcessData.Process.value('(executionStack/frame/@procname)[1]', 'NVARCHAR(255)') AS ProcedureName,
    ProcessData.Process.value('(executionStack/frame/@sqlhandle)[1]', 'NVARCHAR(255)') AS SqlHandle,
    -- Текст последнего запроса в стеке
    SUBSTRING(
        ProcessData.Process.value('(executionStack/frame/@sqlhandle)[1]', 'NVARCHAR(255)'),
        1, 50
    ) AS SqlHandleShort
FROM dbo.DeadlockHistory dh
CROSS APPLY dh.DeadlockGraph.nodes('//deadlock/process-list/process') AS ProcessData(Process)
WHERE dh.Id = @DeadlockId;
```

## 🚨 **Экстренный анализ при активных deadlock'ах**

### **Быстрый мониторинг через системные представления**
```sql
-- Текущие deadlock'ы (из кольцевого буфера)
SELECT 
    DATEADD(mi, DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
           xed.value('@timestamp', 'datetime2')) AS EventTime,
    xed.query('.') AS DeadlockGraph
FROM (
    SELECT CAST(target_data AS XML) AS TargetData
    FROM sys.dm_xe_session_targets st
    INNER JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
    WHERE s.name = 'system_health'
      AND st.target_name = 'ring_buffer'
) AS Data
CROSS APPLY TargetData.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xed)
ORDER BY EventTime DESC;
```

### **Автоматическое оповещение по email**
```sql
-- Триггер на отправку email при новом deadlock'е
CREATE TRIGGER trg_DeadlockAlert
ON dbo.DeadlockHistory
AFTER INSERT
AS
BEGIN
    DECLARE @Count INT, @Body NVARCHAR(MAX);
    
    SELECT @Count = COUNT(*) FROM inserted;
    
    IF @Count > 0
    BEGIN
        SET @Body = N'Обнаружено ' + CAST(@Count AS NVARCHAR) + 
                   N' новых deadlock''ов. ' + CHAR(13) + CHAR(13) +
                   N'Детали: ' + CHAR(13) +
                   (SELECT STRING_AGG(
                        CONCAT(
                            'Время: ', CONVERT(VARCHAR, EventTime, 120),
                            ', БД: ', DatabaseName,
                            ', Приложение: ', Application,
                            ', Объект: ', ISNULL(ContendedObject, 'N/A')
                        ), CHAR(13)
                    ) FROM inserted);
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'YourMailProfile',
            @recipients = 'dba@yourcompany.com',
            @subject = 'SQL Server Deadlock Alert',
            @body = @Body;
    END
END;
```

## 📋 **Checklist для анализа deadlock'а**

1. **Идентифицируйте объект конфликта** (`ContendedObject`)
2. **Определите участвующие приложения** (`Process1_App`, `Process2_App`)
3. **Посмотрите уровни изоляции** (возможно, слишком строгие)
4. **Найдите SQL-запросы** через `sql_handle` (можно расшифровать с помощью `sys.dm_exec_sql_text`)
5. **Проверьте порядок доступа к ресурсам** — основная причина deadlock'ов
6. **Проанализируйте индексы** на конфликтующих таблицах

### **Пример полного анализа конкретного deadlock'а**
```sql
-- 1. Найти deadlock
SELECT TOP 1 DeadlockGraph 
FROM dbo.DeadlockHistory 
WHERE DatabaseName = 'YourProblemDB'
ORDER BY EventTime DESC;

-- 2. Расшифровать sql_handle в текст запроса
DECLARE @SqlHandle VARBINARY(64) = 0x...; -- из DeadlockGraph

SELECT * 
FROM sys.dm_exec_sql_text(@SqlHandle);

-- 3. Проанализировать индексы на проблемной таблице
EXEC sp_helpindex 'ProblemTable';

-- 4. Проверить статистику
DBCC SHOW_STATISTICS('ProblemTable', 'IndexName');
```

## 💡 **Практические советы**

1. **Всегда сохраняйте полный `DeadlockGraph`** — в нем есть вся информация для анализа
2. **Используйте фильтры в сеансе XE**, если deadlock'ов слишком много:
   ```sql
   WHERE ([sqlserver].[database_name] = N'ProblemDB')
   ```
3. **Периодически очищайте старые файлы** `.xel` или настройте `max_rollover_files`
4. **Для Production** установите `STARTUP_STATE = ON`, чтобы сеанс перезапускался автоматически
5. **Храните историю deadlock'ов** минимум 30 дней для выявления паттернов

Эта система даст вам полный контроль над deadlock'ами: от автоматического сбора до детального анализа и оповещений.
