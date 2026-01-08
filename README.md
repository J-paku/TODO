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

```
import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'
import { getSteptaskSyouhin } from '../services/fetchSteptask'
import { useRouter } from 'next/router'
import { ObjectResult, SteptaskItem } from '../types/types'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}` // 'ClassA' ~ 'ClassZ'
type ClassHash = Partial<Record<ClassKey, string>>
type PakuCustomHash = Partial<Record<`Custom${string | number}`, string>>
type DescriptionKey = `Description${Uppercase<string>}` // 'DescriptionA' ~ 'DescriptionZ'
type DescriptionHash = Partial<Record<DescriptionKey, string>>

export interface TouenItem {
  ItemTitle?: string
  タイトル?: string
  UpdatedTime?: string
  updatedTime?: string
  ResultID?: string
  ResultId?: string
  SiteId?: string
  ClassHash?: ClassHash
  PakuCustomHash?: PakuCustomHash
  DescriptionHash?: DescriptionHash
}

interface PreviewRow {
  タイトル?: string
  Title?: string
  更新日時?: string
  UpdatedTime?: string
  updatedTime?: string
}

type ProductsMap = Record<string, Product[]>

type UseTouenItemsParams = {
  /** 最寄り先名（未選択時は null） */
  nearestClientName: string | null
  /** 一度だけ取得されたプレビュー行 */
  previewRowsOnce: PreviewRow[] | null
  customers: Customer[]
  /** 互換維持のため: 本フック内では参照しない（外部状態のみ更新） */
  touenItems?: SteptaskItem[]
  setTouenItems: Dispatch<SetStateAction<SteptaskItem[]>>
  setItemObject: Dispatch<SetStateAction<ObjectResult>>
  setProducts: Dispatch<SetStateAction<Product[]>>
  setProductsMap: Dispatch<SetStateAction<ProductsMap>>
  oneShotPerClient?: boolean
}

type UseTouenItemsReturn = {
  listLoading: boolean
  fetchComplete: boolean
  stableFetchComplete: boolean
  forceRefresh: (clientName?: string) => void
}

/** ---- ユーティリティ ---- */
/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

/** ---- Hook 本体 ---- */
export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary() // 404.tsxにリンクする（/ABC）

  const {
    nearestClientName,
    previewRowsOnce,
    customers,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    oneShotPerClient = true,
  } = params

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  /** 実行済みタイムスタンプを得意先単位で保持（APIでも一度きり制御したい場合用） */
  const ranForClientRef = useRef<Map<string, number>>(new Map())

  /** 再取得の強制（得意先単位 or 全体） */
  const forceRefresh = useCallback(async (clientName?: string) => {
    if (!clientName) return
    setListLoading(true)
    try {
      const apiList: SteptaskItem[] = (await getSteptaskSyouhin(clientName)) ?? []
      setTouenItems(apiList)
      setDataEvaluatedOnce(true)
    } catch (e) {
      showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      console.error(e)
      setDataEvaluatedOnce(true)
    } finally {
      setListLoading(false)
      ranForClientRef.current.set(clientName, Date.now())
    }
  }, [])

  /** fetchComplete がフレーム境界で安定したことを別フラグへ反映 */
  useEffect(() => {
    let rafId: number | null = null
    if (!listLoading && dataEvaluatedOnce) {
      rafId = requestAnimationFrame(() => setStableFetchComplete(true))
    } else {
      setStableFetchComplete(false)
    }
    return () => {
      if (rafId !== null) cancelAnimationFrame(rafId)
    }
  }, [listLoading, dataEvaluatedOnce])

  /** 最寄り先が変わるたびにワンショット制御を解除 */
  useEffect(() => {
    if (nearestClientName) {
      ranForClientRef.current.delete(nearestClientName)
    }
  }, [nearestClientName])

  /** 顧客がロードされたら全体のワンショット制御を解除 */
  useEffect(() => {
    if (customers.length > 0) {
      forceRefresh()
    }
  }, [customers.length, forceRefresh])

  const router = useRouter()

  /** いつでも API → 整合確認 → 画面状態更新（IndexedDBは使用しない） */
  useEffect(() => {
    if (!nearestClientName || nearestClientName === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows: PreviewRow[] = previewRowsOnce ?? []
    const rowForClient = rows.find(r => {
      const title = r?.タイトル ?? r?.Title ?? ''
      return String(title).trim() === String(nearestClientName).trim()
    })
    const currentPreviewTs = ts(
      rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime
    )
    const lastTs = ranForClientRef.current.get(nearestClientName) ?? -1
    if (oneShotPerClient && lastTs >= currentPreviewTs) return

    setListLoading(true)
    let aborted = false

    const errorList: string[] = [] // エラーメッセージArray

    ;(async () => {
      try {
        const apiList: SteptaskItem[] = (await getSteptaskSyouhin(nearestClientName)) ?? []
        if (!aborted) {
          setTouenItems(apiList)
          setDataEvaluatedOnce(true)
        }
      } catch (error) {
        setDataEvaluatedOnce(true)
        errorList.push(String(error)) // エラー文字列積み重ねる
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (!aborted) {
          setListLoading(false)
          ranForClientRef.current.set(nearestClientName, currentPreviewTs)
          if (errorList.length > 0) {
            await logSteptaskErrorCause(errorList.join('\n'))
            showBoundary(
              new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR)
            )
          }
        }
      }
    })()

    return () => {
      aborted = true
      setListLoading(false)
    }
  }, [
    String(nearestClientName),
    previewRowsOnce,
    customers,
    oneShotPerClient,
    setItemObject,
    setTouenItems,
    setProducts,
    setProductsMap,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
'use client'

import { useEffect, useRef, useState } from 'react'
import { useInitializeUser, useSortedCustomersByDistance } from 'hooks/useInitializeUser'
import { useFindNearestClient } from './useFindNearestClient'
import { useFetchCustomerList } from './useFetchCustomerList'
import {
  useUserPositionBundle,
  useUserPositionSync,
  useNearestRecalcOnMove,
} from './useUserPositionBundle'
import { initPreviewAndNearest, PreviewItem } from '../lib/initPreviewAndNearest'
import { extractNameFromMoyoriJson } from '../lib/utils'
import { getInitialProductsForClient } from '../lib/getInitialProductsForClient'
import { Product, UserInfo } from '@/api/loadCustomerData'
import { canUseIndexedDB, getUserInfoCache, probeIndexedDBRoundTrip } from '@/api/db/indexedDB'
import { useLocateAndUpdateNearestClient } from '@/hooks/useLocateAndUpdateNearestClient'
import { ObjectResult, SteptaskItem } from '../types/types'

export const useTouenInitializer = () => {
  // --- 1) ユーザー・顧客・初期状態 ---
  const { userInfo } = useInitializeUser()
  const sessionUserName = userInfo?.Name ?? null

  const { customers, customersByName } = useFetchCustomerList()
  const [moyoriSaki, setMoyoriSaki] = useState<string | null>(null)
  const [seletedClientID, setSeletedClientID] = useState<number | null>(null)

  // --- 2) 位置追跡・並び替え ---
  const {
    liveLat,
    liveLon,
    userLatitude,
    userLongitude,
    setUserLatitude,
    setUserLongitude,
    userPos,
  } = useUserPositionBundle(customers, json => setMoyoriSaki(json))

  let sortLat: number | null = null
  let sortLon: number | null = null

  if (typeof window !== 'undefined' && typeof localStorage !== 'undefined') {
    const latLS = localStorage.getItem('緯度')
    const lonLS = localStorage.getItem('経度')
    sortLat = latLS ? Number(latLS) : null
    sortLon = lonLS ? Number(lonLS) : null
  }

  const sortedItems = useSortedCustomersByDistance(customers, sortLat, sortLon)
  useUserPositionSync(userPos)
  useNearestRecalcOnMove(customers, userPos, setMoyoriSaki)

  // --- 3) 最寄り・手動選択管理 ---
  // ここでは nearestClientName は受け取らず、手動選択APIだけ使う
  const { setManualSelection, forceSelectNearest } = useFindNearestClient(
    customers,
    liveLat,
    liveLon,
    {
      respectManualSelection: true,
      minDistanceChangeMeters: 500,
    }
  )

  // このフック内部で nearestClientName を持つ（initPreviewAndNearest の結果のみを採用）
  const [nearestClientName, _setNearestClientName] = useState<string | null>(null)

  // で確定した最寄り名をロックするフラグ
  // const nearestLockRef = useRef<boolean>(false);

  // 用の setter（init のみが使う。設定と同時にロックON）
  const setNearestClientNameFromInit = (name: string) => {
    const trimmed = String(name)
    _setNearestClientName(trimmed)
    // nearestLockRef.current = true; // 以後は変更を受け付けない

    const obj = customersByName.get(trimmed) ?? null
    void setManualSelection(trimmed, obj) // 画面側の手動選択状態も揃える
    setSeletedClientID(obj?.ID ?? null)
  }

  // 外部（live再計算・ボタン等）からの setter はロック時は無視
  const setNearestClientNameGuarded = (name: string) => {
    // if (nearestLockRef.current) return; // ロック後は何もしない
    const trimmed = String(name)
    _setNearestClientName(trimmed)
    const obj = customersByName.get(trimmed) ?? null
    void setManualSelection(trimmed, obj)
    setSeletedClientID(obj?.ID ?? null)
  }

  // --- 4) 商品・アイテム・プレビュー状態 ---
  const [productsMap, setProductsMap] = useState<Record<string, Product[]>>({})
  const [products, setProducts] = useState<Product[]>([])
  const [touenItems, setTouenItems] = useState<SteptaskItem[]>([])
  const [previewRowsOnce, setPreviewRowsOnce] = useState<PreviewItem[] | null>(null)

  const initialObjectResult: ObjectResult = { ResultID: '', storageCode: '' }
  const [itemObject, setItemObject] = useState<ObjectResult>(initialObjectResult)
  // --- 5) 初期プレビュー・最寄り設定（init のみが最寄りを決定）---
  // データ・位置が準備できてから初回実行
  const isReady =
    Array.isArray(customers) &&
    customers.length > 0 &&
    typeof userLatitude === 'number' &&
    typeof userLongitude === 'number'

  const didInitRef = useRef(false)

  useEffect(() => {
    if (!isReady) return
    if (didInitRef.current) return
    ;(async () => {
      try {
        didInitRef.current = true
        await initPreviewAndNearest({
          userLatitude,
          userLongitude,
          customers,
          touenItems,
          setPreviewRowsOnce,
          setNearestClientName: setNearestClientNameFromInit,
          setMoyoriSaki,
          setItemObject,
          setProducts,
          setProductsMap,
        })
      } catch (e) {
        console.error('プレビュー取得エラー:', e)
        setPreviewRowsOnce([])
      }
    })()
  }, [isReady, customers, userLatitude, userLongitude, touenItems])

  useEffect(() => {
    if (!didInitRef.current && isReady && touenItems.length > 0) {
      ;(async () => {
        try {
          didInitRef.current = true
          await initPreviewAndNearest({
            userLatitude,
            userLongitude,
            customers,
            touenItems,
            setPreviewRowsOnce,
            setNearestClientName: setNearestClientNameFromInit,
            setMoyoriSaki,
            setItemObject,
            setProducts,
            setProductsMap,
          })
        } catch (e) {
          console.error('遅延touenItems取得後プレビューエラー:', e)
          setPreviewRowsOnce([])
        }
      })()
    }
  }, [touenItems, isReady])

  const initialNearestJsonRef = useRef<string | null>(null)
  useEffect(() => {
    if (initialNearestJsonRef.current == null && moyoriSaki) {
      initialNearestJsonRef.current = moyoriSaki
    }
  }, [moyoriSaki])

  // --- 6) 現在位置で即時の最寄り候補再計算 ---
  // init 以外の経路からはロック後に変更させないため、ガード済み setter を渡す
  useLocateAndUpdateNearestClient({
    customers,
    userLatitude: liveLat,
    userLongitude: liveLon,
    setNearestClientName: setNearestClientNameGuarded, // ← 変更
    setUserLatitude,
    setUserLongitude,
  })

  // --- 7) 明示的な最寄り再選定(onLocateNearest) ---
  const onLocateNearest = async () => {
    if (typeof liveLat === 'number' && typeof liveLon === 'number') {
      setUserLatitude(liveLat)
      setUserLongitude(liveLon)
      await new Promise(requestAnimationFrame)
    }

    await forceSelectNearest()

    const name = extractNameFromMoyoriJson(moyoriSaki)
    if (name) {
      const obj = customersByName.get(name) ?? null
      await setManualSelection(name, obj)

      _setNearestClientName(name)
      setSeletedClientID(obj?.ID ?? null)
    }
  }

  // --- 8) 最寄り変更時に初期商品構成 ---
  useEffect(() => {
    const fetchInitialProducts = async () => {
      const isValidClient =
        nearestClientName !== '最寄り先を選択' &&
        nearestClientName !== null &&
        touenItems.length > 0

      if (isValidClient) {
        const defaultProducts = await getInitialProductsForClient(
          nearestClientName!,
          customers,
          touenItems,
          setItemObject
        )

        setProducts(defaultProducts)
        setProductsMap(prev => ({ ...prev, [nearestClientName!]: defaultProducts }))
      }
    }
    fetchInitialProducts()
  }, [nearestClientName, touenItems])

  // --- 9) DB 準備フラグ・キャッシュ有無 ---
  const [isDbReady, setIsDbReady] = useState<boolean>(false)
  const [hasUserInfoCache, setHasUserInfoCache] = useState<boolean>(false)

  useEffect(() => {
    let cancelled = false

    ;(async () => {
      try {
        if (!canUseIndexedDB()) {
          if (!cancelled) {
            setIsDbReady(false)
            setHasUserInfoCache(false)
          }
          return
        }

        const probe = await probeIndexedDBRoundTrip().catch(e => {
          console.warn('IDB プローブ失敗:', e)
          return { ok: false as const }
        })
        if (cancelled) return

        setIsDbReady(!!probe.ok)

        const userInfoCached = await getUserInfoCache<UserInfo>().catch(() => null)
        if (cancelled) return

        setHasUserInfoCache(userInfoCached != null)
      } catch (err) {
        console.warn('DB 準備中のエラー:', err)
        if (!cancelled) {
          setIsDbReady(false)
          setHasUserInfoCache(false)
        }
      }
    })()

    return () => {
      cancelled = true
    }
  }, [])

  // --- 10) フックが返す値 ---
  return {
    // 1) ユーザー関連
    sessionUserName,

    // 2) 顧客・最寄り
    customers,
    customersByName,
    sortedItems,
    moyoriSaki,

    // 由来の最寄り名のみを公開
    nearestClientName,
    // 外部から変えさせないため、公開 setter は「init 専用」のみ必要なら公開
    setNearestClientName: setNearestClientNameFromInit,

    onLocateNearest,

    // 3) 商品・プレビュー
    products,
    setProducts,
    productsMap,
    setProductsMap,
    itemObject,
    setItemObject,
    touenItems,
    setTouenItems,
    previewRowsOnce,

    // 4) 位置
    userLatitude,
    userLongitude,
    liveLat,
    liveLon,

    // 5) DB 状態
    isDbReady,
    hasUserInfoCache,

    // TEST
    seletedClientID,
    setSeletedClientID,
  }
}

```
