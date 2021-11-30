# projekt1
---Dotaz_Trend_zajmu
;WITH
Roky AS
(
	SELECT
		Rok
	FROM
		(VALUES
			(2019),
			(2020),
			(2021)
		) Roky (Rok)
),
Tydny AS (
    SELECT 1 AS Tyden
    UNION ALL
    SELECT Tyden+1 FROM Tydny WHERE Tyden+1<=53
),
ExemplarePoPobockach AS
(
	SELECT
		r.[Rok],
		t.[Tyden],
		Branch,
		e.TitleId,
		tt.Title,
		COUNT(ItemId) AS PocetExemplaru
	FROM
		Roky r
		JOIN Tydny t ON 1=1
		LEFT JOIN exemplare e ON e.AcqDate < DATEADD(WEEK, t.[Tyden], DATEADD(YEAR, r.Rok - 2019, '2019-01-01'))
		LEFT JOIN tituly tt ON tt.TitleId = e.TitleId
	WHERE
		tt.Kind = 'KNI'
		AND e.AcqDate > '2018-01-01'
	GROUP BY
		r.[Rok],
		t.[Tyden],
		Branch,
		e.TitleId,
		tt.Title
),
VypujckyPoPobockach AS
(
	SELECT
		DATEPART(YEAR, v.CheckOutDate) AS Rok,
		DATEPART(WEEK, v.CheckOutDate) AS Tyden,
		v.Branch,
		t.TitleId,
		t.Title,
		COUNT(DISTINCT e.ItemId) AS VypujcenoExemplaru
	FROM
		tituly t
		JOIN exemplare e ON e.TitleId = t.TitleId
		JOIN vypujcky v ON v.ItemId = e.ItemId
	WHERE
		t.Kind = 'KNI'
		AND e.AcqDate > '2019-01-01'
	GROUP BY
		DATEPART(YEAR, v.CheckOutDate),
		DATEPART(WEEK, v.CheckOutDate),
		v.Branch,
		t.TitleId,
		t.Title
),
Pujcovanost AS
(
SELECT
	r.Rok,
	t.Tyden,
	p.ID AS Branch,
	epp.TitleId,
	epp.Title,
	ISNULL(vpp.VypujcenoExemplaru, 0) AS VypujcenoExemplaru,
	ISNULL(epp.PocetExemplaru, 0) AS PocetExemplaru,
	CAST(100.0 * ISNULL(VypujcenoExemplaru, 0) / PocetExemplaru AS int) AS Procentne
FROM
	pobocky p
	JOIN Roky r ON 1=1
	JOIN Tydny t ON 1=1
	LEFT JOIN ExemplarePoPobockach epp ON p.[ID] = epp.[Branch] AND r.[Rok] = epp.[Rok] AND t.[Tyden] = epp.[Tyden]
	LEFT JOIN VypujckyPoPobockach vpp ON p.ID = vpp.[Branch] AND t.Tyden = vpp.Tyden AND r.Rok = vpp.[Rok] AND vpp.[TitleId] = epp.[TitleId]
WHERE
	r.Rok < 2021 OR t.Tyden < 40
)
,
PoLetech AS
(
	SELECT
		Branch,
		Rok,
		TitleId,
		Title,
		SUM(VypujcenoExemplaru) AS VypujcekRocne,
		MAX(PocetExemplaru) AS Exemplaru,
		AVG(Procentne) AS RocniPrumer
	FROM
		Pujcovanost
	GROUP BY
		Branch,
		Rok,
		TitleId,
		Title
), 
Rozdily AS
(
SELECT
	*,
	RocniPrumer - LAG(RocniPrumer) OVER (PARTITION BY Branch, TitleId ORDER BY Rok) AS Rozdil,
	RocniPrumer - LAG(RocniPrumer, 2) OVER (PARTITION BY Branch, TitleId ORDER BY Rok) AS Rozdil2
FROM
	PoLetech
)
SELECT
	[Branch],
	[TitleId],
	[Title],
	[RocniPrumer] AS [Rok2021],
	[Rozdil] AS [Oproti2020],
	[Rozdil2] AS [Oproti2019]
FROM
	Rozdily
WHERE
	Rok = 2021
ORDER BY
	Branch,
	Title,
Rok
