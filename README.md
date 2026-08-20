# ssms-sql-server-management-studio

# Print all tables of all databases.

```sql
EXEC sp_MSforeachdb '
IF ''?'' NOT IN (''tempdb'')
BEGIN
    SELECT
        ''?'' AS DatabaseName,
        TABLE_SCHEMA,
        TABLE_NAME
    FROM ?.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = ''BASE TABLE''
END
';
```

# Search table name in all databases.  

```
EXEC sp_MSforeachdb '
IF ''?'' NOT IN (''master'',''model'',''msdb'',''tempdb'')
BEGIN
    USE [?];
    SELECT
        ''?'' AS DatabaseName,
        SCHEMA_NAME(schema_id) AS SchemaName,
        name AS TableName
    FROM sys.tables
    WHERE name LIKE ''%Cust%'';
END';
```

# Search column name in all databases.

```sql
DECLARE @SearchText SYSNAME = 'CustomerId';

CREATE TABLE #Results (
    DatabaseName SYSNAME,
    SchemaName SYSNAME,
    TableName SYSNAME,
    ColumnName SYSNAME
);

DECLARE @SQL NVARCHAR(MAX) = '';

SELECT @SQL = @SQL + '
USE [' + name + '];

INSERT INTO #Results
SELECT
    DB_NAME(),
    s.name,
    t.name,
    c.name
FROM sys.columns c
JOIN sys.tables t
    ON c.object_id = t.object_id
JOIN sys.schemas s
    ON t.schema_id = s.schema_id
WHERE c.name LIKE ''%' + @SearchText + '%'';
'
FROM sys.databases
WHERE state_desc = 'ONLINE';

EXEC sp_executesql @SQL;

SELECT *
FROM #Results
ORDER BY DatabaseName, SchemaName, TableName;

DROP TABLE #Results;
```

# Table-Valued Parameter TVP

```cs
public class CarType
{
    // Sequence must match SQL Type definition perfectly
    public int CarID { get; set; }
    public string CarName { get; set; }
}

public class CarDto
{
    public int CarID { get; set; }
    public string CarName { get; set; }
}

List<CarType> carList;
DynamicParameters parameters = new DynamicParameters();
parameters.AddTable("@Cars", "[dbo].[CarType]", carList);
List<CarDto> response = new List<VerifyUserDeleteResponseModel>();    
var insertedCars = DatabaseIO.Query<CarDto>("[dbo].[InsertCars]", parameters);
```

```sql
-- 1. Create the Custom Table Type
CREATE TYPE [dbo].[CarType] AS TABLE (
    [CarID] INT NOT NULL,
    [CarName] NVARCHAR(50) NOT NULL
);

-- 2. Create the Stored Procedure using the Table Type

DROP PROCEDURE IF EXISTS [dbo].[InsertCars]
GO

CREATE OR ALTER PROCEDURE [dbo].[InsertCars]
    @Cars [dbo].[CarType] READONLY
AS
BEGIN
    INSERT INTO [dbo].[ActiveInventory] (Id, ModelName)
    OUTPUT INSERTED.Id AS CarID, INSERTED.ModelName AS CarName
    SELECT [CarID], [CarName] FROM @Cars;
END;


```
