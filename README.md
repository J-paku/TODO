```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'
import type { SteptaskItem } from '@/features/touen/types/types'

export type PayloadSteptaskSyouhin = {
  ApiVersion: number
  Offset: number
  PageSize: number
  View: {
    ColumnFilterHash: { Title: string }
    ColumnFilterSearchTypes: { Title: 'ExactMatch' | string }
  }
}

type SteptaskResponseBody = {
  Response?: {
    TotalCount?: number
    Data?: SteptaskItem[]
  }
}

export default async function getSteptaskSyouhin(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<SteptaskResponseBody>> {
  const response = await httpRequest(() =>
    axiosInstance.post<SteptaskResponseBody>(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return {
      code: response.code ?? 500,
      message: response.message,
      data: undefined,
    }
  }

  return {
    code: 200,
    message: 'api get succeeded',
    data: response.data,
  }
}
```

```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse } from '@/api'

import getTouenList, { PayloadGetTouenList } from './list/getTouenList'
import createTouenCount, { PayloadCreateTouenCount } from './count/createTouenCount'
import getTouenClientId from './count/getTouenClientId'
import getSelectedTouenPrice, { PayloadSelectedTouenPrice } from './count/getSelectedTouenPrice'
import deleteSelectedTouen from './list/deleteSelectedTouen'
import { FilterTouenData, FilterTouenDataNotIn, TouenList } from './count/types'
import getTouenStepTaskDataIn, { PayloadGetTouenStepTaskDataIn } from './list/getTouenStepTaskDataIn'
import getTouenStepTaskDataNotIn, {
  PayloadGetTouenStepTaskDataNotIn,
} from './list/getTouenStepTaskDataNotIn'

import getSteptaskSyouhin, { PayloadSteptaskSyouhin } from './count/getSteptaskSyouhin'

type SteptaskResponseBody = {
  Response?: {
    TotalCount?: number
    Data?: unknown[]
  }
}

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  // StepTask 商品
  getSteptaskSyouhin: (payload: PayloadSteptaskSyouhin) => Promise<ApiResponse<SteptaskResponseBody>>

  // 登園リスト
  getTouenList: (
    payload: PayloadGetTouenList<FilterTouenData | FilterTouenDataNotIn>
  ) => Promise<ApiResponse<TouenList[]>>
  getTouenStepTaskDataIn: (payload: PayloadGetTouenStepTaskDataIn) => Promise<ApiResponse<string>>
  getTouenStepTaskDataNotIn: (
    payload: PayloadGetTouenStepTaskDataNotIn
  ) => Promise<ApiResponse<string>>
  deleteSelectedTouen: (result: string) => Promise<ApiResponse<string>>
}

export default function touen(httpRequest: HttpRequest): TouenApi {
  return {
    // 登園カウント
    createTouenCount: payload => createTouenCount(httpRequest, payload),
    getTouenClientId: resultId => getTouenClientId(httpRequest, resultId),
    getSelectedTouenPrice: payload => getSelectedTouenPrice(httpRequest, payload),

    // StepTask 商品
    getSteptaskSyouhin: payload => getSteptaskSyouhin(httpRequest, payload),

    // 登園リスト
    getTouenList: payload => getTouenList(httpRequest, payload),
    getTouenStepTaskDataIn: payload => getTouenStepTaskDataIn(httpRequest, payload),
    getTouenStepTaskDataNotIn: payload => getTouenStepTaskDataNotIn(httpRequest, payload),
    deleteSelectedTouen: resultId => deleteSelectedTouen(httpRequest, resultId),
  }
}
```

```
// サービスレイヤー
import { useMemo } from 'react'
import useHttp from '@/hooks/useHttp'
import useToast from '@/hooks/useToast'
import useSteptaskCreateErrorLog from '@/hooks/useSteptaskCreateErrorLog'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'

import { PayloadCreateTouenCount } from '@/api/touen/count/createTouenCount'
import { PayloadSelectedTouenPrice } from '@/api/touen/count/getSelectedTouenPrice'
import { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

type SteptaskResponseBody = {
  Response?: {
    TotalCount?: number
    Data?: unknown[]
  }
}

export default function useTouenCount() {
  const { api } = useHttp()
  const { openToast } = useToast()
  const { showBoundary } = useErrorBoundary()
  const { createErrorLog } = useSteptaskCreateErrorLog()

  const createTouenCount = useMemo(
    () => async (payload: PayloadCreateTouenCount) => {
      const response = await api.touen.createTouenCount(payload)
      if (response.code !== 200) {
        await createErrorLog(
          'createTouenCount',
          response._error ?? response.message,
          JSON.stringify(payload)
        )
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_MASTER_BULKUPSERT_ERROR))
      }
      return response
    },
    [api, createErrorLog, showBoundary]
  )

  const getTouenClientId = useMemo(
    () => async (resultId: string) => {
      const response = await api.touen.getTouenClientId(resultId)
      if (response.code !== 200) {
        openToast.error(response.message, 'center')
        return undefined
      }
      return response.data
    },
    [api, openToast]
  )

  const getSelectedTouenPrice = useMemo(
    () => async (payload: PayloadSelectedTouenPrice) => {
      const response = await api.touen.getSelectedTouenPrice(payload)
      if (response.code !== 200) {
        openToast.error(response.message, 'center')
        return undefined
      }
      return response.data
    },
    [api, openToast]
  )

  // “POST 1回”だけ返す（ページング＆加工は useFetchTouenItems 側で実施）
  const getSteptaskSyouhin = useMemo(
    () => async (payload: PayloadSteptaskSyouhin): Promise<SteptaskResponseBody | undefined> => {
      const response = await api.touen.getSteptaskSyouhin(payload)
      if (response.code !== 200) {
        openToast.error(response.message, 'center')
        return undefined
      }
      return response.data
    },
    [api, openToast]
  )

  return { createTouenCount, getTouenClientId, getSelectedTouenPrice, getSteptaskSyouhin }
}

```

```
'use client'

import { useCallback, useEffect, useMemo, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'
import type { ObjectResult, SteptaskItem } from '../types/types'
import useTouenCount from './useTouenCountActions'

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}`
type ClassHash = Partial<Record<ClassKey, string>>
type PakuCustomHash = Partial<Record<`Custom${string | number}`, string>>
type DescriptionKey = `Description${Uppercase<string>}`
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
  nearestClientName: string | null
  previewRowsOnce: PreviewRow[] | null
  customers: Customer[]
  touenItems?: SteptaskItem[] // 互換維持（参照しない）
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
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}
const isValidClientName = (name: string | null): name is string =>
  !!name && name !== '最寄り先を選択' && String(name).trim() !== ''

/**
 * ここが「昔 getSteptaskSyouhin 内にあった加工」をそのまま置く場所です。
 * 既にプロジェクト内にある同等実装があるなら import して差し替えてください。
 */
type Detail = {
  koyamaPrice: string | number | null
  destinationCode: string
  productName: string
  productCode: string
  sokoCode: string
  productIndex: number
}

/** ↓↓↓ この4つは “以前の実装” をそのまま使ってください（importに置き換え推奨） */
import { fetchDetail, normalizePrice, alphabetClass, numberClass } from '../services/fetchSteptaskHydrator'
/** ↑↑↑ パスはあなたの実プロジェクトに合わせてください */

const hydrateSyouhinItems = async (list: SteptaskItem[]): Promise<SteptaskItem[]> => {
  for (const item of list) {
    item.PakuCustomHash ||= {}
    ;(item as any).PakuCustomHashTwo ||= {}
    ;(item as any).PakuCustomHashThree ||= {}
    ;(item as any).PakuCustomHashFour ||= {}
    ;(item as any).PakuCustomSoko ||= {}
    item.ClassHash ||= {}
    ;(item as any).PakuCustomHashProductIndex ||= {}
    ;(item as any).PakuCustomHashMasterIndex ||= {}

    const classHash = item.ClassHash as Record<string, string | undefined>

    const alphaFirstKeys = alphabetClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })
    const alphaSecondKeys = numberClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })

    const alphabetMasterIndex =
      alphabetClass
        .map((k, idx) => ({ idx, v: classHash?.[k] }))
        .filter(({ v }) => String(v ?? '').trim() !== '')
        .at(-1)?.idx ?? -1

    const numberMasterIndex =
      numberClass
        .map((k, idx) => ({ idx, v: classHash?.[k] }))
        .filter(({ v }) => String(v ?? '').trim() !== '')
        .at(-1)?.idx ?? -1

    const testAlphaFirstKeys = alphabetClass.filter((_, idx) => idx <= alphabetMasterIndex)
    const testAlphaSecondKeys = numberClass.filter((_, idx) => idx <= numberMasterIndex)

    const draftPakuCustomHash: Record<string, string | null> = {}
    const draftPakuCustomHashTwo: Record<string, string> = {}
    const draftPakuCustomHashThree: Record<string, string> = {}
    const draftPakuCustomHashFour: Record<string, string> = {}
    const draftClassHash: Record<string, string> = {}
    const draftPakuCustomHashProductIndex: Record<string, number> = {}
    const draftPakuCustomHashMasterIndex: Record<string, number> = {}

    const tasksForItem: Promise<void>[] = []

    const alphaFirstCount = alphaFirstKeys.length

    // 英字側（CustomA〜）
    alphaFirstKeys.forEach((key, idx) => {
      const value = classHash[key]
      if (!value) return
      const customKey = `Custom${String.fromCharCode(65 + idx)}`
      tasksForItem.push(
        (async () => {
          const d: Detail = await fetchDetail(value, idx)
          draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
          draftPakuCustomHashTwo[customKey] = d.destinationCode
          draftPakuCustomHashThree[customKey] = d.productName
          draftPakuCustomHashFour[customKey] = d.productCode
          draftClassHash[customKey] = d.sokoCode
          draftPakuCustomHashProductIndex[customKey] = d.productIndex
        })()
      )
    })

    // 数字側（Custom001〜）
    alphaSecondKeys.forEach((key, idx) => {
      const value = classHash[key]
      if (!value) return
      const customKey = `Custom${String(idx + 1).padStart(3, '0')}`
      const productIdx = alphaFirstCount + idx
      tasksForItem.push(
        (async () => {
          const d: Detail = await fetchDetail(value, productIdx)
          draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
          draftPakuCustomHashTwo[customKey] = d.destinationCode
          draftPakuCustomHashThree[customKey] = d.productName
          draftPakuCustomHashFour[customKey] = d.productCode
          draftClassHash[customKey] = d.sokoCode
          draftPakuCustomHashProductIndex[customKey] = d.productIndex
        })()
      )
    })

    // MasterIndex付番
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
      const value = classHash[key]
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
      const value = classHash[key]
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
    Object.assign((item as any).PakuCustomHashTwo, draftPakuCustomHashTwo)
    Object.assign((item as any).PakuCustomHashThree, draftPakuCustomHashThree)
    Object.assign((item as any).PakuCustomHashFour, draftPakuCustomHashFour)
    Object.assign(item.ClassHash!, draftClassHash)
    Object.assign((item as any).PakuCustomHashProductIndex, draftPakuCustomHashProductIndex)
    Object.assign((item as any).PakuCustomHashMasterIndex, draftPakuCustomHashMasterIndex)
  }

  return list
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()
  const { getSteptaskSyouhin } = useTouenCount()

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

  const ranForClientRef = useRef<Map<string, number>>(new Map())

  const buildPayload = useCallback(
    (clientName: string, offset: number, pageSize: number): PayloadSteptaskSyouhin => ({
      ApiVersion: 1.1,
      Offset: offset,
      PageSize: pageSize,
      View: {
        ColumnFilterHash: { Title: clientName },
        ColumnFilterSearchTypes: { Title: 'ExactMatch' },
      },
    }),
    []
  )

  const fetchAllPagesAndHydrate = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      // 1) TotalCount
      const initial = await getSteptaskSyouhin(buildPayload(clientName, 0, 1))
      const totalCount = initial?.Response?.TotalCount
      if (typeof totalCount !== 'number') {
        throw new Error('TotalCount is invalid')
      }

      // 2) 全ページ取得
      const pageSize = 200
      const requests: Promise<ReturnType<typeof getSteptaskSyouhin>>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        requests.push(getSteptaskSyouhin(buildPayload(clientName, offset, pageSize)))
      }
      const pages = await Promise.all(requests)

      const rawList = pages.flatMap(p => {
        const data = p?.Response?.Data
        return Array.isArray(data) ? (data as SteptaskItem[]) : []
      })

      // 3) 加工（昔のロジック）
      const hydrated = await hydrateSyouhinItems(rawList)

      return hydrated
    },
    [buildPayload, getSteptaskSyouhin]
  )

  const forceRefresh = useCallback(
    async (clientName?: string) => {
      const target = clientName ?? nearestClientName ?? null
      if (!isValidClientName(target)) return

      setListLoading(true)
      try {
        const list = await fetchAllPagesAndHydrate(target)
        setTouenItems(list)
        setDataEvaluatedOnce(true)
      } catch (e) {
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
        console.error(e)
        setDataEvaluatedOnce(true)
      } finally {
        setListLoading(false)
        ranForClientRef.current.set(target, Date.now())
      }
    },
    [fetchAllPagesAndHydrate, nearestClientName, setTouenItems, showBoundary]
  )

  // fetchComplete 안정화
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

  // 最寄り先 변경 시, 원샷 제어 해제
  useEffect(() => {
    if (isValidClientName(nearestClientName)) {
      ranForClientRef.current.delete(nearestClientName)
    }
  }, [nearestClientName])

  // customers 로드 후 최초 자동 fetch (원래 동작)
  useEffect(() => {
    if (customers.length > 0 && isValidClientName(nearestClientName)) {
      void forceRefresh(nearestClientName)
    }
  }, [customers.length, nearestClientName, forceRefresh])

  // previewRowsOnce 기반 원샷 제어 (원래 동작)
  useEffect(() => {
    if (!isValidClientName(nearestClientName)) return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows: PreviewRow[] = previewRowsOnce ?? []
    const rowForClient = rows.find(r => {
      const title = r?.タイトル ?? r?.Title ?? ''
      return String(title).trim() === String(nearestClientName).trim()
    })

    const currentPreviewTs = ts(rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime)
    const lastTs = ranForClientRef.current.get(nearestClientName) ?? -1
    if (oneShotPerClient && lastTs >= currentPreviewTs) return

    let aborted = false
    setListLoading(true)

    ;(async () => {
      try {
        const list = await fetchAllPagesAndHydrate(nearestClientName)
        if (!aborted) {
          setTouenItems(list)
          setDataEvaluatedOnce(true)
          ranForClientRef.current.set(nearestClientName, currentPreviewTs)
        }
      } catch (e) {
        if (!aborted) {
          setDataEvaluatedOnce(true)
          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
          console.error(e)
        }
      } finally {
        if (!aborted) setListLoading(false)
      }
    })()

    return () => {
      aborted = true
      setListLoading(false)
    }
  }, [
    nearestClientName,
    previewRowsOnce,
    customers,
    oneShotPerClient,
    fetchAllPagesAndHydrate,
    setTouenItems,
    showBoundary,
    setItemObject,
    setProducts,
    setProductsMap,
  ])

  const fetchComplete = useMemo(() => !listLoading && dataEvaluatedOnce, [listLoading, dataEvaluatedOnce])

  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
