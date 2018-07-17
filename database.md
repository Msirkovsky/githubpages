# Práce s polyglotím dynamem

## Základní info 

Vývojové nástroje:
- PostgreSQL -[PG Admin](https://www.pgadmin.org)
- Oracle - [SQL Developer](https://www.oracle.com/database/technologies/appdev/sql-developer.html)
- [Dbeaver](https://dbeaver.io/) - imho lepší jak oficiální klienti

**Rozdíly:**

[Porovnání vlastností databází](http://www.sql-workbench.net/dbms_comparison.html)

**Ranking:**

[MS SQL vs PostgreSQL](https://db-engines.com/en/system/Microsoft+SQL+Server%3BPostgreSQL)

[MS SQL vs Oracle](https://db-engines.com/en/system/Microsoft+SQL+Server%3BOracle)

Typy v Oracle:
https://docs.oracle.com/cd/B28359_01/server.111/b28318/datatype.htm#CNCPT213

Typy v PostgreSQL
https://www.postgresql.org/docs/current/static/datatype.html

Největší problém UUID a boolean pro Oracle.

# Práce s daty

## Středníky pro Oracle

Pokud se volá SQL query přímo z kódu:
```csharp
_context.Database.SqlQuery<AnalytickyKomentarInfo>("select * from… ")
```

Neukončovat středníky.


### Náhrada za isnull
`isnull` v oracle je `NVL` nebo `coalesce` a v Postgre `coalesce` - ideálně používat všude coalesce kvůli přenositelnosti. Navíc `NVL` je z osmdesátých let a není v SQL normě, stejně jako `isnull`.

Rozdíl mezi `isnull` a `coalesce`. [Odkaz](http://www.itprotoday.com/software-development/coalesce-vs-isnull)


### Oracle nemá tzv. implicitní selekt do paměti typu:

MSSQL/PostgreSQL
```sql
select '1' as cislo
```

Oracle

```sql
select '1' as cislo from dual
```

dual je tabulka v paměti, která se na to "zneužívá"

### Varchar(max)

**Oracle:**

Vyhýbat se `clob`(Oraclí alternativa k `varchar(max)`). Nelze filtrovat podle clobu. Bohužel varchar v Oracle má omezení:

- varchar(MAX) -> varchar2(4000)
- nvarchar(MAX) -> nvarchar2(2000)

varchar2 je alias pro varchar. Doporučuje se používat varchar2 - varchar se v budoucnu může změnit.

**Postgre**
- varchar(MAX) -> varchar/text
- nvarchar(MAX) -> varchar/text

`Varchar` má omezení na **1GB**. `Text` žádné


### Newid()

Oracle:
```SQL
select sys_guid() as s from dual
```

Postgre: 
```SQL
select uuid_generate_v4()`
```

Pozn. pro Oracle, pokud to někdy nepůjde, je nutné jednorázově zavolat:
```SQL
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### Literál UUID
Pro Oracle nelze jednoduše použít literál uniqueidentifier jako má MSSQL/PostgreSQL v selektech - nutné použití `HEXTORAW`:

**MSSQL/Postgre:**
```
select  ak.Id                            
from dbo.AnalytickyKomentar ak
join (select '80160AF5-2A5E-4FCC-9E8C-41C699792FBC' as Id)M on M.Id = ak.ObjectId
```

**Oracle**
```
select  ak.Id                            
from dbo.AnalytickyKomentar ak
 join (select HEXTORAW('7002F96F440048E9E0535A02000AB256') as Id from dual)M on M.Id = ak.ObjectId
```


 
 ### Výraz: colName ?? string.Empty 
 
 EF pro Oracle tento zápis neumí správně přeložit do SQL:

```C#
... 
join tm in context.Set<TypMethod>() on m.TypMethodId equals tm.Id into tmTemp from tmX in tmTemp.DefaultIfEmpty()
                    select new MethodInfo()
                    {                        
                        //toto spadne
                        TypMetody = tmX.Name ?? string.Empty
                    }).ToList();
```

## Dynamo

### Psaní selektů
Nutné mít všude prefixy i pro DBO!

### Metoda pro získání typu databáze

```csharp
Entity.DbEngineType.Get()
```


Definuje se ve web.configu:
```xml
<add key="ASDSoft.DynaApp.DbEngine" value="Oracle"/>
```
Možné hodnoty: `Oracle`,`Postgre`, `MSSQL`
Viz `Entity.DatabaseType`

```
public enum DatabaseType
{
    MSSQL = 0,
    Postgre = 1,
    Oracle = 2
}
```

Pozn. Podle toho lze měnit selekty v kódu.  Ukázka použití je v:

```csharp 
AnalytickyKomentarRepository.GetAnalytickyKomentarInfoSelect
```

Převod GUID do Oracle (pomocná extension metoda ToOracleUUID())

Příklad:
```csharp 
Guid.NewGuid().ToOracleUUID() //vrací string, který zvládne Oracle
```

### Registrace závislostí ve statických metodách:
```C#
CrossDomainProxy.RegisterOracle
CrossDomainProxy.RegisterPostgre
CrossDomainProxy.RegisterMSSQL
```

Např.

`ILoggingService` je jiná pro všechny tři databáze, takže existují tři třídy a jsou registrované v metodách podle DB.


<br/><br/><br/><br/>
## A na další rozdíly se ještě narazí časem :(


