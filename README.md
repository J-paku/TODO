```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'

/**
 * StepTask "ITEMS_GET" のレスポンス最小形（必要に応じて拡張）
 * - initialRes.data.Response.TotalCount
 * - initialRes.data.Response.Data
 * を壊さないために、Response をそのまま返す
 */
export type StepTaskItemsGetResponse<T = unknown> = {
  Response?: {
    TotalCount?: number
    Data?: T[]
  }
}

/**
 * payload は「呼び出し側(useFetchTouenItems等)で組み立てる」前提
 * ここでは shape を固定しすぎず、プロジェクトの実態に合わせて柔軟に扱う
 */
export type PayloadSteptaskSyouhin = {
  ApiVersion?: number
  ApiKey?: string
  Offset?: number
  PageSize?: number
  View?: {
    ColumnFilterHash?: {
      Title?: string
      [key: string]: unknown
    }
    ColumnFilterSearchTypes?: {
      Title?: string
      [key: string]: unknown
    }
    [key: string]: unknown
  }
  [key: string]: unknown
}

export default async function getSteptaskSyouhin<T = unknown>(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<StepTaskItemsGetResponse<T>>> {
  const response = await httpRequest(() =>
    axiosInstance.post(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return {
      code: response.code ?? 500,
      message: response.message,
      data: undefined,
    }
  }

  // 重要: StepTask の生レスポンス構造をそのまま返す（TotalCount/Data を壊さない）
  return {
    code: 200,
    message: 'api get succeeded',
    data: response.data as StepTaskItemsGetResponse<T>,
  }
}

```

```
import type { HttpRequest } from '@/hooks/useHttp'
import type { ApiResponse } from '@/api'

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
  StepTaskItemsGetResponse,
} from './count/getSteptaskSyouhin'

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  /**
   * StepTask 商品取得:
   * TotalCount / Data を使用するため、Response全体を返す
   */
  getSteptaskSyouhin: <T = unknown>(
    payload: PayloadSteptaskSyouhin
  ) => Promise<ApiResponse<StepTaskItemsGetResponse<T>>>

  // 登園リスト
  getTouenList: (
    payload: PayloadGetTouenList<FilterTouenData | FilterTouenDataNotIn>
  ) => Promise<ApiResponse<TouenList[]>>

  getTouenStepTaskDataIn: (payload: PayloadGetTouenStepTaskDataIn) => Promise<ApiResponse<string>>
  getTouenStepTaskDataNotIn: (
    payload: PayloadGetTouenStepTaskDataNotIn
  ) => Promise<ApiResponse<string>>

  deleteSelectedTouen: (resultId: string) => Promise<ApiResponse<string>>
}

export default function touen(httpRequest: HttpRequest): TouenApi {
  return {
    // 登園カウント
    createTouenCount: payload => createTouenCount(httpRequest, payload),
    getTouenClientId: resultId => getTouenClientId(httpRequest, resultId),
    getSelectedTouenPrice: payload => getSelectedTouenPrice(httpRequest, payload),

    // StepTask 商品取得
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
import { useMemo } from 'react'
import useHttp from '@/hooks/useHttp'
import useToast from '@/hooks/useToast'
import useSteptaskCreateErrorLog from '@/hooks/useSteptaskCreateErrorLog'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'

import { PayloadCreateTouenCount } from '@/api/touen/count/createTouenCount'
import { PayloadSelectedTouenPrice } from '@/api/touen/count/getSelectedTouenPrice'
import {
  PayloadSteptaskSyouhin,
  StepTaskItemsGetResponse,
} from '@/api/touen/count/getSteptaskSyouhin'

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

  /**
   * StepTask 商品取得（TotalCount/Data が必要なので Response全体を返す）
   * 呼び出し側では:
   *   const res = await getSteptaskSyouhin<SteptaskItem>(payload)
   *   const total = res?.Response?.TotalCount
   *   const data = res?.Response?.Data ?? []
   */
  const getSteptaskSyouhin = useMemo(
    () => async <T = unknown>(payload: PayloadSteptaskSyouhin) => {
      const response = await api.touen.getSteptaskSyouhin<T>(payload)
      if (response.code !== 200) {
        openToast.error(response.message, 'center')
        return undefined
      }
      return response.data as StepTaskItemsGetResponse<T>
    },
    [api, openToast]
  )

  return { createTouenCount, getTouenClientId, getSelectedTouenPrice, getSteptaskSyouhin }
}

```
