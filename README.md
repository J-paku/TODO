```
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'
import steptaskAxios from '@/api/steptaskAxios'
import { handleApiError, isDebug, ApiKey } from '@/api/fetchSteptask'
import { SteptaskParams } from '@/api/types/api'
import { ERR_PREFIX, postWithRetry } from './retry'
import { alphabetClass, numberClass } from './classesConstants'
import { fetchDetail } from './steptaskDetailService'
import { normalizePrice } from './format'
import PQueue from 'p-queue'

const requestQueue = new PQueue({ concurrency: 5 })

// 得意先比較(更新日時・totalConut)
export const TouenCleintPreviewCount = async () => {
  try {
    const url = TOUEN_API_ENDPOINTS.MASTER_GET
    const GridColumns = ['UpdatedTime']

    // 最初のリウエストはPageSize = 1
    const initialParams: SteptaskParams = {
      ApiVersion: 1.1,
      Offset: 0,
      PageSize: 1,
      ColumnSorterHash: { UpdatedTime: 'desc' },
      View: {
        ApiDataType: 'KeyValues',
        GridColumns,
        ColumnFilterHash: {
          ClassZ: '["登園セット（個人請求）","登園セット（稼働請求）","登園セット"]',
          ClassD: '["導入済","導入済（営業進捗報告あり）"]',
          Description001: ' ',
        },
        ColumnFilterNegatives: ['Description001'],
      },
    }

    if (isDebug) initialParams.ApiKey = ApiKey

    const initialRes = await steptaskAxios.post(url, initialParams)

    const totalCount = initialRes.data.Response?.TotalCount || 0
    const updatedTime = initialRes.data.Response.Data?.[0].更新日時

    return { totalCount, updatedTime }
  } catch (error) {
    handleApiError(error)
  }
}

// プレビュー
export const TouenItemsPreviewCount = async () => {
  try {
    const pageSize = 200
    const url = TOUEN_API_ENDPOINTS.ITEMS_GET
    const params: SteptaskParams = {
      ApiVersion: 1.1,
      View: {
        PageSize: pageSize,
        ApiDataType: 'KeyValues',
        GridColumns: ['UpdatedTime', 'Title'],
        ColumnSorterHash: { UpdatedTime: 'desc' },
      },
    }

    if (isDebug) params.ApiKey = ApiKey

    const firstRes = await steptaskAxios.post(url, { ...params, Offset: 0 })
    const firstData = firstRes.data.Response.Data
    const totalCount = firstRes.data?.Response?.TotalCount ?? firstData.length

    // Offsetリスト生成
    const offsets: number[] = []
    for (let offset = pageSize; offset < totalCount; offset += pageSize) {
      offsets.push(offset)
    }

    const results = await Promise.all(
      offsets.map(offset =>
        requestQueue.add(() =>
          steptaskAxios.post(url, { ...params, Offset: offset }, { timeout: 180000 })
        )
      )
    )

    const allData = [...firstData, ...results.flatMap(res => res.data.Response.Data)]

    return { row: allData, totalCount }
  } catch (error) {
    handleApiError(error)
    return { row: [], totalCount: 0 }
  }
}
// 指定された得意先に関連する商品情報を取得する関数
// 失敗時はthrowして全体中断（Fail-fast）
export const getSteptaskSyouhin = async (tokuisaki: string) => {
  const url = TOUEN_API_ENDPOINTS.ITEMS_GET
  const pageSize = 200

  try {
    // 初回：総件数確認
    const initialParams: SteptaskParams = {
      ApiVersion: 1.1,
      Offset: 0,
      PageSize: 1,
      View: {
        ColumnFilterHash: { Title: tokuisaki },
        ColumnFilterSearchTypes: { Title: 'ExactMatch' },
      },
    }
    if (isDebug) initialParams.ApiKey = ApiKey

    const initialRes = await postWithRetry(url, initialParams)
    const totalCount = initialRes.data.Response?.TotalCount
    if (typeof totalCount !== 'number') {
      throw new Error(`${ERR_PREFIX}TotalCountが不正です`)
    }

    // 全ページ取得（部分成功なし）
    const pageRequests: Promise<Awaited<ReturnType<typeof postWithRetry>>>[] = []
    for (let offset = 0; offset < totalCount; offset += pageSize) {
      const params = { ...initialParams, Offset: offset, PageSize: pageSize }
      pageRequests.push(postWithRetry(url, params))
    }
    const pageResponses = await Promise.all(pageRequests)
    const list = pageResponses.flatMap(res => res.data?.Response?.Data ?? [])

    // アイテム単位の加工（ドラフト→単一コミット）
    for (const item of list) {
      item.PakuCustomHash ||= {}
      item.PakuCustomHashTwo ||= {}
      item.PakuCustomHashThree ||= {}
      item.PakuCustomHashFour ||= {}
      item.PakuCustomSoko ||= {}
      item.ClassHash ||= {}
      item.PakuCustomHashProductIndex ||= {}
      item.PakuCustomHashMasterIndex ||= {}

      // 値ありキー抽出
      const alphaFirstKeys = alphabetClass.filter(k => {
        const v = item.ClassHash?.[k]
        return v !== undefined && v !== null && String(v).trim() !== ''
      })
      const alphaSecondKeys = numberClass.filter(k => {
        const v = item.ClassHash?.[k]
        return v !== undefined && v !== null && String(v).trim() !== ''
      })

      // 末尾値ありインデックス
      const alphabetMasterIndex =
        alphabetClass
          .map((k, idx) => ({ idx, v: item.ClassHash?.[k] }))
          .filter(({ v }) => String(v ?? '').trim() !== '')
          .at(-1)?.idx ?? -1

      const numberMasterIndex =
        numberClass
          .map((k, idx) => ({ idx, v: item.ClassHash?.[k] }))
          .filter(({ v }) => String(v ?? '').trim() !== '')
          .at(-1)?.idx ?? -1

      const testAlphaFirstKeys = alphabetClass.filter((_, idx) => idx <= alphabetMasterIndex)
      const testAlphaSecondKeys = numberClass.filter((_, idx) => idx <= numberMasterIndex)

      // ドラフト
      const draftPakuCustomHash: Record<string, string | null> = {}
      const draftPakuCustomHashTwo: Record<string, string> = {}
      const draftPakuCustomHashThree: Record<string, string> = {}
      const draftPakuCustomHashFour: Record<string, string> = {}
      const draftClassHash: Record<string, string> = {}
      const draftPakuCustomHashProductIndex: Record<string, number> = {}
      const draftPakuCustomHashMasterIndex: Record<string, number> = {}

      const tasksForItem: Promise<void>[] = []

      // 英字側（CustomA〜）
      const alphaFirstCount = alphaFirstKeys.length
      alphaFirstKeys.forEach((key, idx) => {
        const value = item.ClassHash![key]
        const customKey = `Custom${String.fromCharCode(65 + idx)}`
        tasksForItem.push(
          (async () => {
            const d = await fetchDetail(value, idx)
            draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
            draftPakuCustomHashTwo[customKey] = d.destinationCode
            draftPakuCustomHashThree[customKey] = d.productName
            draftPakuCustomHashFour[customKey] = d.productCode
            draftClassHash[customKey] = d.sokoCode
            draftPakuCustomHashProductIndex[customKey] = d.productIndex
          })()
        )
      })

      // 数字側（Custom001〜）※ 連番継続: alphaFirstCount + idx
      alphaSecondKeys.forEach((key, idx) => {
        const value = item.ClassHash![key]
        const customKey = `Custom${String(idx + 1).padStart(3, '0')}`
        const productIdx = alphaFirstCount + idx
        tasksForItem.push(
          (async () => {
            const d = await fetchDetail(value, productIdx)
            draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
            draftPakuCustomHashTwo[customKey] = d.destinationCode
            draftPakuCustomHashThree[customKey] = d.productName
            draftPakuCustomHashFour[customKey] = d.productCode
            draftClassHash[customKey] = d.sokoCode
            draftPakuCustomHashProductIndex[customKey] = d.productIndex
          })()
        )
      })

      // MasterIndex付番（A,B,C...は1〜10、001,002...は11〜）
      const createAlphaKeyGen = () => {
        let i = 0
        return () => `Custom${String.fromCharCode(65 + i++)}`
      }
      const createNumericKeyGen = () => {
        let i = 1
        return () => `Custom${String(i++).padStart(3, '0')}`
      }
      const nextAlphaKey = createAlphaKeyGen()
      const nextNumericKey = createNumericKeyGen()

      testAlphaFirstKeys.forEach((key, idx) => {
        const value = item.ClassHash![key]
        if (String(value ?? '').trim() === '') return
        const customKey = nextAlphaKey()
        const masterIndex = idx + 1
        tasksForItem.push(
          (async () => {
            draftPakuCustomHashMasterIndex[customKey] = masterIndex
          })()
        )
      })

      testAlphaSecondKeys.forEach((key, idx) => {
        const value = item.ClassHash![key]
        if (String(value ?? '').trim() === '') return
        const customKey = nextNumericKey()
        const masterIndex = idx + 11
        tasksForItem.push(
          (async () => {
            draftPakuCustomHashMasterIndex[customKey] = masterIndex
          })()
        )
      })

      await Promise.all(tasksForItem)

      Object.assign(item.PakuCustomHash!, draftPakuCustomHash)
      Object.assign(item.PakuCustomHashTwo!, draftPakuCustomHashTwo)
      Object.assign(item.PakuCustomHashThree!, draftPakuCustomHashThree)
      Object.assign(item.PakuCustomHashFour!, draftPakuCustomHashFour)
      Object.assign(item.ClassHash!, draftClassHash)
      Object.assign(item.PakuCustomHashProductIndex!, draftPakuCustomHashProductIndex)
      Object.assign(item.PakuCustomHashMasterIndex!, draftPakuCustomHashMasterIndex)
    }

    return list
  } catch (err) {
    const e = err instanceof Error ? err : new Error(`${ERR_PREFIX}不明なエラー`)
    console.error(e)
    throw e
  }
}

```
