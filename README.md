# StroredProcedures
stored procedures

USE [OilCompany]
GO
/****** Object:  StoredProcedure [dbo].[RatingAZS]    Script Date: 01.08.2016 9:40:28 ******/
SET ANSI_NULLS ON
	
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Author,,Name>
-- Create date: <Create Date,,>
-- Description:	<Description,,>
-- =============================================

CREATE PROCEDURE [dbo].[RaitingAZS] 
	(
		@FromDate date,
		@ToDate date
	)
AS
BEGIN
	SET NOCOUNT ON;
	SELECT 
		   CASE 
				WHEN fact.rating  IS NULL THEN '-'
				ELSE CAST(fact.rating AS NVARCHAR(20)) 
		   end as 'Рейтинг АЗС',
		   CASE
				WHEN gs.Title IS NULL AND GROUPING_ID(SUBSTRING(gs.Title,1,1)) = 1 THEN 'Общий результат' 
				WHEN gs.Title IS NULL THEN 'Итого' 
				ELSE gs.Title
		   END AS 'Код АЗС', 
		   CASE
				WHEN b.Title IS NULL THEN '-'
				ELSE b.Title
		   END AS 'Филиал',
		   CASE
				WHEN r.Title IS NULL THEN '-'
				ELSE r.Title
		   END AS 'Область',
		   SUM(fact.SumFact) AS 'Факт реализаций, тонн',
		   SUM(splan.SumPlan) AS 'План реализаций, тонн'
	FROM [GasStation] gs
	LEFT JOIN [Branch] b ON b.ID = gs.BranchID
	LEFT JOIN [Region] r ON r.id = gs.RegionID	
    JOIN(
		SELECT 
			sf.GasStationID,  
			SUM(sf.SalesTones) AS SumFact,
			ROW_NUMBER() OVER (PARTITION by SUBSTRING(g.Title, 1, 1) ORDER BY (SELECT(g.Title))) AS rating
		FROM [SalesFact] sf
		LEFT JOIN [GasStation] g ON g.ID = sf.GasStationID
		WHERE sf.SalesDate between @FromDate and @ToDate
		GROUP BY sf.GasStationID, g.Title
		) fact on fact.GasStationID = gs.ID
	LEFT JOIN (
		SELECT sp.GasStationID, SUM(sp.Amount) AS SumPlan
		FROM SalesPlan sp
		WHERE sp.PlanMonth between @FromDate and @ToDate	
		GROUP BY sp.GasStationID
		) splan ON splan.GasStationID = gs.ID 
   GROUP BY
		GROUPING SETS(
			(fact.rating, SUBSTRING(gs.Title, 1, 1), gs.Title, b.Title, r.Title), 
			(SUBSTRING(gs.Title, 1, 1)), 
			() 
		)
END

