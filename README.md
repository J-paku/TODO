DECLARE @会計年度 NVARCHAR(10) = '2024';
DECLARE @取引先コード NVARCHAR(15) = '1324400100';
DECLARE @取引先名 NVARCHAR(15) = 'シーティープランニング';

-- First query
SELECT
    会社コード, 
    会計年度,
    NULL AS 伝票日付, -- Adding this to match the second query
    FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月,
    部門コード,
    部門名称,
    科目コード,
    CASE 
        WHEN 科目名称 = '未払金' THEN '消耗品費 他'
        ELSE 科目名称
    END AS 科目名称,
    伝票NO,
    検索NO,
    取引先コード,
    取引先名,
    摘要,
    貸方金額,
    借方金額,
    CASE 
        WHEN 科目コード = '41200' AND 相手科目コード IN ('81352', '82855') AND 伝票NO = 伝票NO THEN '82855'
        ELSE 相手科目コード
    END AS 相手科目コード,
    相手科目名称,
    NULL AS 消費税コード,
    NULL AS 消費税率,
    CASE 
        WHEN 相手科目コード = '3999' THEN 0
        ELSE 貸方金額
    END AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV 
WHERE
    会社コード = '9999'
    AND 会計年度 = @会計年度
    AND 取引先コード = @取引先コード
    AND 貸方金額 IS NOT NULL
    AND 科目コード <> '41200'

UNION ALL

-- Second query
SELECT
    会社コード, 
    会計年度,
    伝票日付,
    FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月,
    部門コード,
    部門名称,
    科目コード,
    CASE 
        WHEN 科目名称 = '未払金' THEN '消耗品費 他'
        ELSE 科目名称
    END AS 科目名称,
    伝票NO,
    検索NO,
    取引先コード,
    取引先名,
    摘要,
    貸方金額,
    借方金額,
    CASE 
        WHEN 科目コード = '41200' AND 相手科目コード IN ('81352', '82855') AND 伝票NO = 伝票NO THEN '82855'
        ELSE 相手科目コード
    END AS 相手科目コード,
    相手科目名称,
    ISNULL(消費税コード, 0) AS 消費税コード,
    CASE 
        WHEN 消費税コード = 20 THEN 0 
        ELSE ISNULL(消費税率, 0) 
    END AS 消費税率,
    CAST(KV.[借方金額] * (1 + CASE 
        WHEN 消費税コード = 20 THEN 0 
        ELSE ISNULL(消費税率, 0) 
    END / 100) AS INT) AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV 
WHERE
    FORMAT(KV.[伝票日付], 'yyyy/MM') = '2024/08'
    AND 伝票NO = '30000176'
    AND 相手科目コード = '3999'
    --AND 借方金額 is not null

ORDER BY 伝票年月, 取引先名 DESC;

위 쿼리는 8월달의 相手科目コード = '3999'인 데이터의 伝票NO와 FORMAT(KV.[伝票日付], 'yyyy/MM')로 where을 하고 있는데

아래 쿼리의 伝票NO와 FORMAT(KV.[伝票日付], 'yyyy/MM')를 동적으로 하고 싶어
2024/04의 相手科目コード = '3999'데이터의 伝票NO를 쓰는 그런느낌으로 4월부터 5월 6월까지 있는 데
