This is the instructions to setup Fabric environment for this workshop https://github.com/larsbi/DynUG-Fabric-Workshop-V2.

It requires to create a Fabric SQL database and a Fabric Lakehouse

**Fabric SQL database**
Create SQL Database in Fabric
Make sure all required users have access to the database

Load Sample data
-> You get SalesLT schema

**Create DimCustomer table**
```sql
DROP TABLE [SalesLT].[DimCustomer]
GO
CREATE TABLE [SalesLT].[DimCustomer](
	[CustomerID] [int] NOT NULL,
	[FirstName] [nvarchar](50) NULL,
	[LastName] [nvarchar](50) NULL,
	[CompanyName] [nvarchar](128) NULL,
	[SalesPerson] [nvarchar](256) NULL
) ON [PRIMARY]
GO

insert into SalesLT.DimCustomer
select CustomerId, FirstName, LastName, CompanyName, SalesPerson
from SalesLT.Customer
GO
```

**Create DimDate table**
```sql
CREATE TABLE [SalesLT].[DimDate](
	[DateID] [int] NOT NULL,
	[FullDateAlternateKey] [date] NOT NULL,
	[DayNumberOfWeek] [tinyint] NOT NULL,
	[DayNameOfWeek] [nvarchar](10) NOT NULL,
	[DayNumberOfMonth] [tinyint] NOT NULL,
	[DayNumberOfYear] [smallint] NOT NULL,
	[WeekNumberOfYear] [tinyint] NOT NULL,
	[MonthName] [nvarchar](10) NOT NULL,
	[MonthNumberOfYear] [tinyint] NOT NULL,
	[CalendarQuarter] [tinyint] NOT NULL,
	[CalendarYear] [smallint] NOT NULL
) ON [PRIMARY]
GO

DECLARE @StartDate DATETIME = '01/01/2025' --Starting value of Date Range
DECLARE @EndDate DATETIME = '12/31/2025' --End Value of Date Range

DECLARE @CurrentDate AS DATETIME = @StartDate

WHILE @CurrentDate < @EndDate
BEGIN

/* Populate Your Dimension Table with values*/
	
	INSERT INTO SalesLT.[DimDate]
	SELECT
		
		CONVERT (char(8),@CurrentDate,112) as DateID,
		@CurrentDate AS FullDateAlternateKey,
		-- check for day of week as Per US and change it as per UK format 
		CASE DATEPART(DW, @CurrentDate)
			WHEN 1 THEN 7
			WHEN 2 THEN 1
			WHEN 3 THEN 2
			WHEN 4 THEN 3
			WHEN 5 THEN 4
			WHEN 6 THEN 5
			WHEN 7 THEN 6
			END 
			AS DayNumberOfWeek,
		DATENAME(DW, @CurrentDate) AS DayNameOfWeek,
		DATEPART(DD, @CurrentDate) AS DayNumberOfMonth,
		DATEPART(DY, @CurrentDate) AS DayNumberOfYear,
		DATEPART(WW, @CurrentDate) AS WeekNumberOfYear,
		DATENAME(MM, @CurrentDate) AS MonthName,		
		DATEPART(MM, @CurrentDate) AS MonthNumberOfYear,
		DATEPART(QQ, @CurrentDate) AS CalendarQuarter,
		DATEPART(YEAR, @CurrentDate) AS CalendarYear

	SET @CurrentDate = DATEADD(DD, 1, @CurrentDate)
END
```

**Create SalesOrderHeaderDW table**
```sql
CREATE TABLE [SalesLT].[SalesOrderHeaderDW](
	[SalesOrderID] [int] NOT NULL,
	[RevisionNumber] [tinyint] NOT NULL,
	[OrderDate] [datetime] NOT NULL,
	[DueDate] [datetime] NOT NULL,
	[ShipDate] [datetime] NULL,
	[Status] [tinyint] NOT NULL,
	[OnlineOrderFlag] [bit] NULL,
	[PurchaseOrderNumber] [nvarchar](25) NULL,
	[AccountNumber] [nvarchar](15) NULL,
	[CustomerID] [int] NOT NULL,
	[ShipToAddressID] [int] NULL,
	[BillToAddressID] [int] NULL,
	[ShipMethod] [nvarchar](50) NOT NULL,
	[CreditCardApprovalCode] [varchar](15) NULL,
	[SubTotal] [money] NOT NULL,
	[TaxAmt] [money] NOT NULL,
	[Freight] [money] NOT NULL,
	[Comment] [nvarchar](max) NULL,
	[rowguid] [uniqueidentifier] NOT NULL,
	[ModifiedDate] [datetime] NOT NULL,
	[StoreID] [int] NULL
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
GO

Execute twice:
INSERT INTO SalesLT.SalesOrderHeaderDW 
    (SalesOrderID, RevisionNumber, OrderDate, DueDate, ShipDate, Status, OnlineOrderFlag, 
     PurchaseOrderNumber, AccountNumber, CustomerID, ShipToAddressID, BillToAddressID, 
     ShipMethod, CreditCardApprovalCode, SubTotal, TaxAmt, Freight, Comment, rowguid, ModifiedDate)
SELECT 
    SalesOrderID, RevisionNumber, OrderDate, DueDate, ShipDate, Status, OnlineOrderFlag, 
    PurchaseOrderNumber, AccountNumber, CustomerID, ShipToAddressID, BillToAddressID, 
    ShipMethod, CreditCardApprovalCode, SubTotal, TaxAmt, Freight, Comment, rowguid, ModifiedDate
FROM SalesLT.SalesOrderHeader;

UPDATE SalesLT.SalesOrderHeaderDW
SET StoreID = 1
WHERE CustomerID between 29485 and 29612;

UPDATE SalesLT.SalesOrderHeaderDW
SET StoreID = 2
WHERE CustomerID between 29613 and 29847;

UPDATE SalesLT.SalesOrderHeaderDW
SET StoreID = 3
WHERE CustomerID between 29848 and 30027;

UPDATE SalesLT.SalesOrderHeaderDW
SET StoreID = 4
WHERE CustomerID > 30027;
```

**Create DimProduct View**
```sql
CREATE VIEW [SalesLT].[DimProduct]
AS
select
P.ProductID,
P.Name,
P.ProductNumber,
P.StandardCost,
P.ListPrice,
PC.Name AS CategoryName
FROM SalesLT.Product P
JOIN SalesLT.ProductCategory PC
ON P.ProductCategoryID = PC.ProductCategoryID
GO
```

**Create FactSales View**
```sql
CREATE VIEW [SalesLT].[FactSales] 
AS 
select 
CAST(FORMAT(SOH.OrderDate, 'yyyyMMdd') as Int) DateID, 
SOH.CustomerID,
SOD.ProductID,
SOH.StoreID,
SOD.OrderQty, 
sum(SOD.OrderQty * SOD.UnitPrice) TotalSales
from SalesLT.SalesOrderHeaderDW SOH
join  SalesLT.SalesOrderDetail SOD 
ON SOH.SalesOrderID = SOD.SalesOrderID
group by SOH.OrderDate, SOH.CustomerID, SOH.StoreID,
SOD.OrderQty, SOD.ProductID
GO
```

**Fabric Lakehouse**
Create Lakehouse with schema in Fabric
Make sure all required users have access to the database

Upload file [Customer reviews.xlsx](https://github.com/larsbi/Fabric-Workshop-Setup/blob/main/Customer%20reviews.xlsx) to the root folder in Lakehouse
