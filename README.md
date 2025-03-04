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