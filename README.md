WITH CTE AS (
    -- 4월~6월 동안 `相手科目コード = '3999'` 인 데이터에서 `伝票NO`, `伝票日付` 추출
    SELECT DISTINCT 伝票NO, FORMAT(伝票日付, 'yyyy/MM') AS 伝票年月
    FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳]
    WHERE FORMAT(伝票日付, 'yyyy/MM') BETWEEN '2024/04' AND '2024/06'
      AND 相手科目コード = '3999'
)

-- 추출한 `伝票NO`, `伝票日付` 를 활용하여 데이터 조회
SELECT 
    KV.会社コード, KV.会計年度, KV.伝票日付, 
    FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月, 
    KV.部門コード, KV.部門名称, KV.科目コード, 
    CASE 
        WHEN KV.科目名称 = '未払金' THEN '消耗品費 他' 
        ELSE KV.科目名称 
    END AS 科目名称, 
    KV.伝票NO, KV.検索NO, KV.取引先コード, KV.取引先名, 
    KV.摘要, KV.貸方金額, KV.借方金額, 
    CASE 
        WHEN KV.科目コード = '41200' AND KV.相手科目コード IN ('81352', '82855') 
        THEN '82855' 
        ELSE KV.相手科目コード 
    END AS 相手科目コード, 
    KV.相手科目名称, 
    ISNULL(KV.消費税コード, 0) AS 消費税コード, 
    CASE 
        WHEN KV.消費税コード = 20 THEN 0 
        ELSE ISNULL(KV.消費税率, 0) 
    END AS 消費税率, 
    CAST(KV.[借方金額] * (1 + CASE WHEN KV.消費税コード = 20 THEN 0 ELSE ISNULL(KV.消費税率, 0) END / 100) AS INT) AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV
-- 4~6월 동안 `相手科目コード = '3999'` 인 전표 데이터를 기반으로 필터링
INNER JOIN CTE ON KV.伝票NO = CTE.伝票NO
               AND FORMAT(KV.伝票日付, 'yyyy/MM') = CTE.伝票年月
WHERE KV.相手科目コード = '3999'
ORDER BY 伝票年月, 取引先名 DESC;




ㅅㅐㄱ스

DECLARE @会計年度 NVARCHAR(10) = '2024'; -- 유저가 입력한 회계연도
DECLARE @開始年月 NVARCHAR(7);
DECLARE @終了年月 NVARCHAR(7);

-- 해당 회계연도의 4월부터 다음 해 3월까지 범위 설정
SET @開始年月 = @会計年度 + '/04';
SET @終了年月 = CAST(CAST(@会計年度 AS INT) + 1 AS NVARCHAR(10)) + '/03';

WITH CTE AS (
    -- `相手科目コード = '3999'` 인 데이터에서 `伝票NO`, `伝票日付` 가져오기 (회계연도 반영)
    SELECT DISTINCT 伝票NO, FORMAT(伝票日付, 'yyyy/MM') AS 伝票年月
    FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳]
    WHERE FORMAT(伝票日付, 'yyyy/MM') BETWEEN @開始年月 AND @終了年月
      AND 相手科目コード = '3999'
)

-- 동적으로 전표번호와 전표일자를 적용한 메인 쿼리
SELECT 
    KV.会社コード, KV.会計年度, KV.伝票日付, 
    FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月, 
    KV.部門コード, KV.部門名称, KV.科目コード, 
    CASE 
        WHEN KV.科目名称 = '未払金' THEN '消耗品費 他' 
        ELSE KV.科目名称 
    END AS 科目名称, 
    KV.伝票NO, KV.検索NO, KV.取引先コード, KV.取引先名, 
    KV.摘要, KV.貸方金額, KV.借方金額, 
    CASE 
        WHEN KV.科目コード = '41200' AND KV.相手科目コード IN ('81352', '82855') 
        THEN '82855' 
        ELSE KV.相手科目コード 
    END AS 相手科目コード, 
    KV.相手科目名称, 
    ISNULL(KV.消費税コード, 0) AS 消費税コード, 
    CASE 
        WHEN KV.消費税コード = 20 THEN 0 
        ELSE ISNULL(KV.消費税率, 0) 
    END AS 消費税率, 
    CAST(KV.[借方金額] * (1 + CASE WHEN KV.消費税コード = 20 THEN 0 ELSE ISNULL(KV.消費税率, 0) END / 100) AS INT) AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV
-- 회계연도 4월~3월의 `相手科目コード = '3999'` 데이터를 기반으로 필터링
INNER JOIN CTE ON KV.伝票NO = CTE.伝票NO
               AND FORMAT(KV.伝票日付, 'yyyy/MM') = CTE.伝票年月
WHERE KV.相手科目コード = '3999'
ORDER BY 伝票年月, 取引先名 DESC;


새ㄱ스

DECLARE @会計年度 NVARCHAR(10) = '2024';
DECLARE @取引先コード NVARCHAR(15) = '1324400100';
DECLARE @取引先名 NVARCHAR(15) = 'シーティープランニング';
DECLARE @開始年月 NVARCHAR(7);
DECLARE @終了年月 NVARCHAR(7);

-- 해당 회계연도의 4월부터 다음 해 3월까지 범위 설정
SET @開始年月 = @会計年度 + '/04';
SET @終了年月 = CAST(CAST(@会計年度 AS INT) + 1 AS NVARCHAR(10)) + '/03';

WITH CTE AS (
    -- 해당 회계연도의 4월~3월 사이에 `相手科目コード = '3999'` 데이터를 가져옴
    SELECT DISTINCT 伝票NO, FORMAT(伝票日付, 'yyyy/MM') AS 伝票年月
    FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳]
    WHERE FORMAT(伝票日付, 'yyyy/MM') BETWEEN @開始年月 AND @終了年月
      AND 相手科目コード = '3999'
)

-- 두 개의 쿼리를 합침
SELECT 会社コード, 会計年度, NULL AS 伝票日付,
       FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月,
       部門コード, 部門名称, 科目コード,
       CASE WHEN 科目名称 = '未払金' THEN '消耗品費 他' ELSE 科目名称 END AS 科目名称,
       伝票NO, 検索NO, 取引先コード, 取引先名, 摘要, 貸方金額, 借方金額,
       CASE WHEN 科目コード = '41200' AND 相手科目コード IN ('81352', '82855') AND 伝票NO = 伝票NO THEN '82855' ELSE 相手科目コード END AS 相手科目コード,
       相手科目名称, NULL AS 消費税コード, NULL AS 消費税率,
       CASE WHEN 相手科目コード = '3999' THEN 0 ELSE 貸方金額 END AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV
WHERE 会社コード = '9999' 
  AND 会計年度 = @会計年度
  AND 取引先コード = @取引先コード
  AND 貸方金額 IS NOT NULL
  AND 科目コード <> '41200'
  AND 伝票NO IN (SELECT 伝票NO FROM CTE)

UNION ALL

SELECT 会社コード, 会計年度, 伝票日付,
       FORMAT(KV.[伝票日付], 'yyyy/MM') AS 伝票年月,
       部門コード, 部門名称, 科目コード,
       CASE WHEN 科目名称 = '未払金' THEN '消耗品費 他' ELSE 科目名称 END AS 科目名称,
       伝票NO, 検索NO, 取引先コード, 取引先名, 摘要, 貸方金額, 借方金額,
       CASE WHEN 科目コード = '41200' AND 相手科目コード IN ('81352', '82855') AND 伝票NO = 伝票NO THEN '82855' ELSE 相手科目コード END AS 相手科目コード,
       相手科目名称, ISNULL(消費税コード, 0) AS 消費税コード,
       CASE WHEN 消費税コード = 20 THEN 0 ELSE ISNULL(消費税率, 0) END AS 消費税率,
       CAST(KV.[借方金額] * (1 + CASE WHEN 消費税コード = 20 THEN 0 ELSE ISNULL(消費税率, 0) END / 100) AS INT) AS 税込金額
FROM [Integration].[dbo].[会計_V_部門別科目別補助元帳] AS KV
WHERE 伝票NO IN (SELECT 伝票NO FROM CTE)
  AND 相手科目コード = '3999'
ORDER BY 伝票年月, 取引先名 DESC;