```
import axios, { AxiosInstance } from 'axios'
import { z } from 'zod'

export type ApiResponse<T> = {
  pagination?: {
    Offset: number
    PageSize: number
    TotalCount: number
  }
  data: T | null | undefined
  message: string
  code: number | string
  _error?: z.core.$ZodIssue[]
}

// this const is load on App start should be careful
export const axiosInstance: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_PLEASANTER_URL, //base url config here or env
  // headers: {
  //   Accept: '*/*', // for test
  //   'Access-Control-Allow-Origin': '*', // for test
  //   'Content-Type': 'application/json',
  // },
})
// Request Interceptor
axiosInstance.interceptors.request.use(
  config => {
    config.params = {
      ...config.params,
      // version: '1.0'
    }
    if (config.method === 'post' || config.method === 'put') {
      config.data = {
        ...config.data,
        ...(process.env.NODE_ENV === 'development' && {
          Apikey: process.env.NEXT_PUBLIC_API_PLEASANTER_API_KEY,
        }),
        ApiVersion: '1.1',
      }
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)

// Response Interceptor
axiosInstance.interceptors.response.use(
  response => {
    return response
  },
  async error => {
    if (error.response && error.response.data) {
      const contentType = error.response.header['content-type']
      if (contentType && contentType.includes('application/json')) {
        try {
          const text = await error.response.data.text() // Convert Blob to Text for handle error
          const json = JSON.parse(text)
          error.response.data = json
        } catch (err) {
          console.error('Failed to parse JSON :', err)
        }
      }
    }
    return Promise.reject(error)
  }
)

```

```
import type { HttpRequest } from '@/hooks/useHttp'
import type { ApiResponse } from '@/api'

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
import type { SteptaskItem } from '@/features/touen/types/types' // 프로젝트 실제 경로에 맞춰 조정하세요

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  // ★ StepTask 商品（ITEMS_GET）
  getSteptaskSyouhin: (payload: PayloadSteptaskSyouhin) => Promise<ApiResponse<SteptaskItem[]>>

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

    // ★ StepTask
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
import { PayloadCreateTouenCount } from '@/api/touen/count/createTouenCount'
import { PayloadSelectedTouenPrice } from '@/api/touen/count/getSelectedTouenPrice'
import useHttp from '@/hooks/useHttp'
import useToast from '@/hooks/useToast'
import useSteptaskCreateErrorLog from '@/hooks/useSteptaskCreateErrorLog'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'
import getSteptaskSyouhinPayloadType, {
  PayloadSteptaskSyouhin,
} from '@/api/touen/count/getSteptaskSyouhin'
import type { SteptaskItem } from '@/features/touen/types/types' // 프로젝트 경로에 맞게 조정

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
        return
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
        return
      }
      return response.data
    },
    [api, openToast]
  )

  /**
   * ★ ここでは「1回分の ITEMS_GET」を返すだけ
   * 　全ページ取得・加工は useFetchTouenItems 側に寄せる
   */
  const getSteptaskSyouhin = useMemo(
    () => async (payload: PayloadSteptaskSyouhin) => {
      const response = await api.touen.getSteptaskSyouhin(payload)
      if (response.code !== 200) {
        // 必要ならここで toast を出す。ただし自動エラーを嫌うなら黙る方が安全
        // openToast.error(response.message, 'center')
        return null
      }
      return response
    },
    [api]
  )

  return { createTouenCount, getTouenClientId, getSelectedTouenPrice, getSteptaskSyouhin }
}

```

```
'use client'

import { useCallback, useEffect, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import useTouenCount from '@/features/touen/hooks/useTouenCountActions' // 실제 경로로 조정

import type { ObjectResult, SteptaskItem } from '@/features/touen/types/types' // 실제 경로로 조정
import type { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

// 필요 시: 기존 로직 유지용(상품 초기화)
// 경로는 프로젝트에 맞게 조정하세요.
import { getInitialProductsForClient } from '@/features/touen/lib/getInitialProductsForClient'

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

/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()
  const { getSteptaskSyouhin } = useTouenCount()

  const {
    nearestClientName,
    previewRowsOnce,
    customers,
    touenItems,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    oneShotPerClient = true,
  } = params

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  /** 実行済みタイムスタンプを得意先単位で保持 */
  const ranForClientRef = useRef<Map<string, number>>(new Map())

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

  /**
   * ★ 1クライアント分を「TotalCount 取得 → 全ページ取得 → 結合」して返す
   *   （旧 fetchSteptask 側の initialRes / totalCount ループをここに移動）
   */
  const fetchAllPagesForClient = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

      // 1) TotalCount確認（PageSize=1）
      const initialPayload: PayloadSteptaskSyouhin = {
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: clientName },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initialRes = await getSteptaskSyouhin(initialPayload)
      if (!initialRes || initialRes.code !== 200) {
        throw new Error('Initial ITEMS_GET failed')
      }

      const totalCount = initialRes.pagination?.TotalCount ?? 0
      if (!Number.isFinite(totalCount)) {
        throw new Error('TotalCount is invalid')
      }

      // TotalCount=0
      if (totalCount <= 0) return []

      // 2) 全ページ取得
      const tasks: Promise<SteptaskItem[]>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        const payload: PayloadSteptaskSyouhin = {
          ...initialPayload,
          Offset: offset,
          PageSize: pageSize,
        }
        tasks.push(
          (async () => {
            const res = await getSteptaskSyouhin(payload)
            if (!res || res.code !== 200) return []
            return res.data ?? []
          })()
        )
      }

      const pages = await Promise.all(tasks)
      return pages.flat()
    },
    [getSteptaskSyouhin]
  )

  /**
   * 再取得の強制（得意先単位）
   * index.tsx から forceRefresh(nearestClientName) を呼べるようにする
   */
  const forceRefresh = useCallback(
    async (clientName?: string) => {
      if (!clientName) return
      setListLoading(true)
      try {
        const list = await fetchAllPagesForClient(clientName)
        setTouenItems(list)
        setDataEvaluatedOnce(true)

        // 旧ロジック維持: items から products / itemObject を生成
        if (customers.length > 0) {
          const products = await getInitialProductsForClient(
            clientName,
            customers,
            touenItems ?? [],
            obj => setItemObject(obj),
            list // ★ override に “今回取得した list” を渡すのが重要
          )
          setProducts(products)
          setProductsMap(prev => ({ ...prev, [clientName]: products }))
        }
      } catch (e) {
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
        console.error(e)
        setDataEvaluatedOnce(true)
      } finally {
        setListLoading(false)
        ranForClientRef.current.set(clientName, Date.now())
      }
    },
    [
      customers,
      fetchAllPagesForClient,
      setItemObject,
      setProducts,
      setProductsMap,
      setTouenItems,
      showBoundary,
      touenItems,
    ]
  )

  /**
   * nearestClientName 変更時の自動取得（one-shot 制御あり）
   * previewRowsOnce の更新日時と ranForClientRef を比較して抑制
   */
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

    let aborted = false
    setListLoading(true)

    ;(async () => {
      try {
        const list = await fetchAllPagesForClient(nearestClientName)
        if (aborted) return

        setTouenItems(list)
        setDataEvaluatedOnce(true)
        ranForClientRef.current.set(nearestClientName, currentPreviewTs)

        // 旧ロジック維持: items から products / itemObject を生成
        const products = await getInitialProductsForClient(
          nearestClientName,
          customers,
          touenItems ?? [],
          obj => setItemObject(obj),
          list
        )
        if (aborted) return
        setProducts(products)
        setProductsMap(prev => ({ ...prev, [nearestClientName]: products }))
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
    fetchAllPagesForClient,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    showBoundary,
    touenItems,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
