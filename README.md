```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'
import type { SteptaskItem } from '@/lib/commonTouen/types/types' // 프로젝트 경로에 맞게 조정

export type StepTaskView = {
  ColumnFilterHash: { Title: string }
  ColumnFilterSearchTypes: { Title: 'ExactMatch' | string }
}

export type PayloadSteptaskSyouhin = {
  ApiVersion: number
  Offset: number
  PageSize: number
  View: StepTaskView
  ApiKey?: string
}

export type StepTaskItemsResponse = {
  Response?: {
    TotalCount?: number
    Data?: SteptaskItem[]
  }
}

export default async function getSteptaskSyouhin(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<StepTaskItemsResponse>> {
  const response = await httpRequest(() =>
    axiosInstance.post<StepTaskItemsResponse>(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return {
      code: Number(response.code ?? 500),
      message: response.message,
      data: undefined,
      _error: response,
    }
  }

  return {
    code: 200,
    message: 'api get success',
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

import type { FilterTouenData, FilterTouenDataNotIn, TouenList } from './count/types'

import getTouenStepTaskDataIn, { PayloadGetTouenStepTaskDataIn } from './list/getTouenStepTaskDataIn'
import getTouenStepTaskDataNotIn, {
  PayloadGetTouenStepTaskDataNotIn,
} from './list/getTouenStepTaskDataNotIn'

import getSteptaskSyouhin, {
  PayloadSteptaskSyouhin,
  StepTaskItemsResponse,
} from './count/getSteptaskSyouhin'

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  // StepTask 商品取得（※ページネーション・加工はHook側で実施）
  getSteptaskSyouhin: (payload: PayloadSteptaskSyouhin) => Promise<ApiResponse<StepTaskItemsResponse>>

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

    // StepTask
    getSteptaskSyouhin: payload => getSteptaskSyouhin(httpRequest, payload),

    // 登園リスト
    getTouenList: payload => getTouenList(httpRequest, payload),
    getTouenStepTaskDataIn: payload => getTouenStepTaskDataIn(httpRequest, payload),
    getTouenStepTaskDataNotIn: payload => getTouenStepTaskDataNotIn(httpRequest, payload),
    deleteSelectedTouen: resultId => deleteSelectedTouen(httpRequest, resultId),
  }
}

export type { PayloadSteptaskSyouhin, StepTaskItemsResponse }

```

```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse } from '@/api'

import getTouenList, { PayloadGetTouenList } from './list/getTouenList'
import createTouenCount, { PayloadCreateTouenCount } from './count/createTouenCount'
import getTouenClientId from './count/getTouenClientId'
import getSelectedTouenPrice, { PayloadSelectedTouenPrice } from './count/getSelectedTouenPrice'
import deleteSelectedTouen from './list/deleteSelectedTouen'

import type { FilterTouenData, FilterTouenDataNotIn, TouenList } from './count/types'

import getTouenStepTaskDataIn, { PayloadGetTouenStepTaskDataIn } from './list/getTouenStepTaskDataIn'
import getTouenStepTaskDataNotIn, {
  PayloadGetTouenStepTaskDataNotIn,
} from './list/getTouenStepTaskDataNotIn'

import getSteptaskSyouhin, {
  PayloadSteptaskSyouhin,
  StepTaskItemsResponse,
} from './count/getSteptaskSyouhin'

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  // StepTask 商品取得（※ページネーション・加工はHook側で実施）
  getSteptaskSyouhin: (payload: PayloadSteptaskSyouhin) => Promise<ApiResponse<StepTaskItemsResponse>>

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

    // StepTask
    getSteptaskSyouhin: payload => getSteptaskSyouhin(httpRequest, payload),

    // 登園リスト
    getTouenList: payload => getTouenList(httpRequest, payload),
    getTouenStepTaskDataIn: payload => getTouenStepTaskDataIn(httpRequest, payload),
    getTouenStepTaskDataNotIn: payload => getTouenStepTaskDataNotIn(httpRequest, payload),
    deleteSelectedTouen: resultId => deleteSelectedTouen(httpRequest, resultId),
  }
}

export type { PayloadSteptaskSyouhin, StepTaskItemsResponse }

```

```
import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'

import useTouenCount from '@/hooks/useTouenCountActions'
import type { PayloadSteptaskSyouhin } from '@/api/touen'
import type { ObjectResult, SteptaskItem } from '../types/types' // 프로젝트 경로에 맞게 조정

type ProductsMap = Record<string, Product[]>

type PreviewRow = {
  タイトル?: string
  Title?: string
  更新日時?: string
  UpdatedTime?: string
  updatedTime?: string
}

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

const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

// ---- 여기서부터 “fetchStepTask(전체 취득/가공)” 로직을 Hook 안으로 ----
const PAGE_SIZE = 200

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

  // 得意先ごとに「最後に評価した更新TS」を保持
  const ranForClientRef = useRef<Map<string, number>>(new Map())

  // ---- ここが “1得意先ぶんの StepTask 全件取得” ----
  const fetchAllForClient = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      // 1) TotalCount 取得
      const initialPayload: PayloadSteptaskSyouhin = {
        ApiVersion: 1.1,
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: clientName },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initialRes = await getSteptaskSyouhin(initialPayload)
      if (initialRes.code !== 200) {
        throw new Error(`StepTask initial request failed: ${initialRes.message}`)
      }

      const totalCount = initialRes.data?.Response?.TotalCount
      if (typeof totalCount !== 'number') {
        throw new Error('StepTask TotalCount is invalid')
      }

      // 2) 全ページ取得
      const reqs: Promise<ReturnType<typeof getSteptaskSyouhin>>[] = []
      for (let offset = 0; offset < totalCount; offset += PAGE_SIZE) {
        const payload: PayloadSteptaskSyouhin = {
          ...initialPayload,
          Offset: offset,
          PageSize: PAGE_SIZE,
        }
        reqs.push(getSteptaskSyouhin(payload))
      }

      const results = await Promise.all(reqs)

      // 3) 失敗が混じってたら全体を失敗にする（部分成功なし）
      const failed = results.find(r => r.code !== 200)
      if (failed) {
        throw new Error(`StepTask page request failed: ${failed.message}`)
      }

      const list = results.flatMap(r => r.data?.Response?.Data ?? [])

      // 4) ここで「fetchStepTask にあった加工」を実行したいなら、この位置に移す
      //    ※ あなたの既存関数/定数( fetchDetail / normalizePrice / alphabetClass ... ) を
      //       “参照するだけ”でOKなら import してここで適用する。
      //    ※ 今回は「動作を壊さない」ため、最低限 “hash の初期化” だけ入れておく。
      for (const item of list) {
        // any を 쓰지 않기 위해 unknown 기반으로 안전하게 처리
        const it = item as unknown as {
          PakuCustomHash?: Record<string, unknown>
          PakuCustomHashTwo?: Record<string, unknown>
          PakuCustomHashThree?: Record<string, unknown>
          PakuCustomHashFour?: Record<string, unknown>
          PakuCustomSoko?: Record<string, unknown>
          ClassHash?: Record<string, unknown>
          PakuCustomHashProductIndex?: Record<string, unknown>
          PakuCustomHashMasterIndex?: Record<string, unknown>
        }

        it.PakuCustomHash ||= {}
        it.PakuCustomHashTwo ||= {}
        it.PakuCustomHashThree ||= {}
        it.PakuCustomHashFour ||= {}
        it.PakuCustomSoko ||= {}
        it.ClassHash ||= {}
        it.PakuCustomHashProductIndex ||= {}
        it.PakuCustomHashMasterIndex ||= {}
      }

      return list
    },
    [getSteptaskSyouhin]
  )

  // 강제 재조회 (UI 버튼 등에서 사용)
  const forceRefresh = useCallback(
    (clientName?: string) => {
      if (!clientName) return

      setListLoading(true)
      setDataEvaluatedOnce(false)

      ;(async () => {
        try {
          const list = await fetchAllForClient(clientName)
          setTouenItems(list)
          setDataEvaluatedOnce(true)
          ranForClientRef.current.set(clientName, Date.now())
        } catch (e) {
          console.error(e)
          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
          setDataEvaluatedOnce(true)
        } finally {
          setListLoading(false)
        }
      })()
    },
    [fetchAllForClient, setTouenItems, showBoundary]
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

  // nearest 변경 시 one-shot 초기화
  useEffect(() => {
    if (nearestClientName) ranForClientRef.current.delete(nearestClientName)
  }, [nearestClientName])

  // ✅ “원래처럼 자동으로 움직이되” oneShotPerClient 기준으로 효율화
  useEffect(() => {
    if (!nearestClientName || nearestClientName === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows = previewRowsOnce ?? []
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
    const errorList: string[] = []

    ;(async () => {
      try {
        const list = await fetchAllForClient(nearestClientName)
        if (aborted) return

        setTouenItems(list)
        setDataEvaluatedOnce(true)

        // 여기서 필요하면 setItemObject / setProducts / setProductsMap 업데이트 로직을 그대로 유지/이식
        // (당신 프로젝트 기존 로직이 있다면 그 부분만 여기로 가져오면 됩니다)

      } catch (err) {
        console.error(err)
        errorList.push(String(err))
        setDataEvaluatedOnce(true)

        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (aborted) return

        setListLoading(false)
        ranForClientRef.current.set(nearestClientName, currentPreviewTs)

        if (errorList.length > 0) {
          await logSteptaskErrorCause(errorList.join('\n'))
        }
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
    fetchAllForClient,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    showBoundary,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
