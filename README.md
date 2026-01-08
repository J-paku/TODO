```
import axios from 'axios'
import { axiosInstance, ApiResponse } from '@/api'
import type { HttpRequest } from '@/hooks/useHttp'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'

/**
 * Pleasanter 응답 최소 형태 (KeyValues 기반)
 * - 프로젝트마다 컬럼명/형태가 섞일 수 있으므로, 훅에서 후처리(가공)한다.
 */
type PleasanterItemsResponse<T> = {
  Response?: {
    TotalCount?: number
    Data?: T[]
  }
}

export type PayloadSteptaskSyouhin = {
  // axios interceptor가 ApiVersion/ApiKey는 자동 부가하므로 여기서는 View/Offset/PageSize만 넘긴다
  Offset: number
  PageSize: number
  View: {
    ColumnFilterHash: { Title: string }
    ColumnFilterSearchTypes: { Title: string } // 'ExactMatch'
  }
}

/**
 * StepTask: ITEMS_GET (Title ExactMatch)
 * - 이 API 레이어는 "페이지 단위 raw list + pagination"만 반환한다.
 * - 전체 페이지 취득 + item 가공은 훅(useFetchTouenItems)에서 수행한다.
 */
export default async function getSteptaskSyouhin<T = unknown>(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<
  ApiResponse<{
    list: T[]
    pagination: { Offset: number; PageSize: number; TotalCount: number }
  }>
> {
  const response = await httpRequest(() =>
    axiosInstance.post<PleasanterItemsResponse<T>>(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (!response) {
    return { code: 500, message: 'api response is undefined', data: null }
  }

  if (axios.isAxiosError(response)) {
    return {
      code: (response as any).code ?? 500,
      message: response.message,
      data: null,
      _error: response,
    } as any
  }

  const totalCountRaw = response.data?.Response?.TotalCount
  const dataRaw = response.data?.Response?.Data

  const totalCount =
    typeof totalCountRaw === 'number' ? totalCountRaw : Array.isArray(dataRaw) ? dataRaw.length : 0

  const list = Array.isArray(dataRaw) ? dataRaw : []

  const pagination = {
    Offset: payload.Offset,
    PageSize: payload.PageSize,
    TotalCount: totalCount,
  }

  return {
    code: 200,
    message: 'api get successed',
    data: { list, pagination },
    pagination,
  } as any
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

import getSteptaskSyouhin, { PayloadSteptaskSyouhin } from './count/getSteptaskSyouhin'

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>

  /**
   * StepTask: ITEMS_GET (Title ExactMatch)
   * - 훅에서 “전체 페이지 취득 + item 가공”을 수행할 것이므로
   * - 여기서는 페이지 단위 raw list만 반환한다.
   */
  getSteptaskSyouhin: <T = unknown>(
    payload: PayloadSteptaskSyouhin
  ) => Promise<
    ApiResponse<{
      list: T[]
      pagination: { Offset: number; PageSize: number; TotalCount: number }
    }>
  >

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

// あなたのプロジェクトの型
import type { SteptaskItem } from '../types/types'

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
   * StepTask(ITEMS_GET) 페이지 단위 raw fetch
   * - "전체 페이지 취득 + item 가공"은 useFetchTouenItems 훅에서 수행한다.
   * - 원래 로직의 Fail-fast를 살리기 위해, 실패 시 throw 한다.
   */
  const getSteptaskSyouhin = useMemo(
    () => async (payload: PayloadSteptaskSyouhin) => {
      const response = await api.touen.getSteptaskSyouhin<SteptaskItem>(payload)

      if (response.code !== 200 || !response.data) {
        // 로깅(선택): 원래 로직 감각(실패 시 전체 중단)을 유지하되 추적은 남긴다.
        void createErrorLog(
          'getSteptaskSyouhin',
          response._error ?? response.message,
          JSON.stringify(payload)
        )

        // 토스트는 UX에 따라 선택. 여기서는 최소로만.
        openToast.error(response.message, 'center')

        // 핵심: throw (Fail-fast)
        throw new Error(response._error ?? response.message ?? 'getSteptaskSyouhin failed')
      }

      return response.data // { list, pagination }
    },
    [api, createErrorLog, openToast]
  )

  return { createTouenCount, getTouenClientId, getSelectedTouenPrice, getSteptaskSyouhin }
}

```

```
'use client'

import { useCallback, useEffect, useMemo, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'

// 서비스 레이어
import useTouenCount from './useTouenCountActions'
import type { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

// 기존 타입
import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

// 기존 로직(상품 초기화)
import { getInitialProductsForClient } from '../lib/getInitialProductsForClient'

// ====== “원래 로직”에서 쓰던 의존성들 ======
import { alphabetClass, numberClass } from '../services/classesConstants'
import { fetchDetail } from '../services/steptaskDetailService'
import { normalizePrice } from '../services/format'
// ============================================

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}`
type ClassHash = Partial<Record<ClassKey, string>>

type PreviewRow = {
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
  forceRefresh: (clientName?: string) => Promise<void>
}

/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

/**
 * StepTask item 가공(= 원래 getSteptaskSyouhin 내부 로직을 그대로 이관)
 * - 입력 list를 직접 mutate(원래 로직도 mutate)
 * - fetchDetail를 각 item 내에서 병렬 처리 후 단일 커밋(Object.assign)
 */
async function enrichItemsInPlace(list: SteptaskItem[]): Promise<void> {
  for (const item of list) {
    item.PakuCustomHash ||= {}
    item.PakuCustomHashTwo ||= {}
    item.PakuCustomHashThree ||= {}
    item.PakuCustomHashFour ||= {}
    item.PakuCustomSoko ||= {}
    item.ClassHash ||= {}
    item.PakuCustomHashProductIndex ||= {}
    item.PakuCustomHashMasterIndex ||= {}

    const classHash = item.ClassHash as ClassHash

    // 値ありキー抽出
    const alphaFirstKeys = alphabetClass.filter(k => {
      const v = (classHash as any)?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })
    const alphaSecondKeys = numberClass.filter(k => {
      const v = (classHash as any)?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })

    // 末尾値ありインデックス
    const alphabetMasterIndex =
      alphabetClass
        .map((k, idx) => ({ idx, v: (classHash as any)?.[k] }))
        .filter(({ v }) => String(v ?? '').trim() !== '')
        .at(-1)?.idx ?? -1

    const numberMasterIndex =
      numberClass
        .map((k, idx) => ({ idx, v: (classHash as any)?.[k] }))
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
      const value = (classHash as any)[key]
      if (String(value ?? '').trim() === '') return

      const customKey = `Custom${String.fromCharCode(65 + idx)}`
      tasksForItem.push(
        (async () => {
          const d = await fetchDetail(String(value), idx)
          draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
          draftPakuCustomHashTwo[customKey] = d.destinationCode
          draftPakuCustomHashThree[customKey] = d.productName
          draftPakuCustomHashFour[customKey] = d.productCode
          draftClassHash[customKey] = d.sokoCode
          draftPakuCustomHashProductIndex[customKey] = d.productIndex
        })()
      )
    })

    // 数字側（Custom001〜）※連番継続: alphaFirstCount + idx
    alphaSecondKeys.forEach((key, idx) => {
      const value = (classHash as any)[key]
      if (String(value ?? '').trim() === '') return

      const customKey = `Custom${String(idx + 1).padStart(3, '0')}`
      const productIdx = alphaFirstCount + idx
      tasksForItem.push(
        (async () => {
          const d = await fetchDetail(String(value), productIdx)
          draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
          draftPakuCustomHashTwo[customKey] = d.destinationCode
          draftPakuCustomHashThree[customKey] = d.productName
          draftPakuCustomHashFour[customKey] = d.productCode
          draftClassHash[customKey] = d.sokoCode
          draftPakuCustomHashProductIndex[customKey] = d.productIndex
        })()
      )
    })

    // MasterIndex付番（A,B,C...は1〜、001,002...は11〜）
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
      const value = (classHash as any)[key]
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
      const value = (classHash as any)[key]
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

    // 단일 커밋(원래 로직 그대로)
    Object.assign(item.PakuCustomHash!, draftPakuCustomHash)
    Object.assign(item.PakuCustomHashTwo!, draftPakuCustomHashTwo)
    Object.assign(item.PakuCustomHashThree!, draftPakuCustomHashThree)
    Object.assign(item.PakuCustomHashFour!, draftPakuCustomHashFour)

    // 주의: 원래 로직은 item.ClassHash에 sokoCode를 Custom키로 덮어씀
    Object.assign(item.ClassHash!, draftClassHash)

    Object.assign(item.PakuCustomHashProductIndex!, draftPakuCustomHashProductIndex)
    Object.assign(item.PakuCustomHashMasterIndex!, draftPakuCustomHashMasterIndex)
  }
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()

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

  const { getSteptaskSyouhin } = useTouenCount()

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  /** 実行済みタイムスタンプを得意先単位で保持（previewTs 기준） */
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

  const getPreviewTsForClient = useCallback(
    (clientName: string) => {
      const rows: PreviewRow[] = previewRowsOnce ?? []
      const rowForClient = rows.find(r => {
        const title = r?.タイトル ?? r?.Title ?? ''
        return String(title).trim() === String(clientName).trim()
      })
      return ts(rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime)
    },
    [previewRowsOnce]
  )

  const markRan = useCallback(
    (clientName: string, fallback?: number) => {
      const pTs = getPreviewTsForClient(clientName)
      ranForClientRef.current.set(clientName, pTs || fallback || Date.now())
    },
    [getPreviewTsForClient]
  )

  /**
   * ページ全取得(Title ExactMatch) → item 가공
   * - 원래 로직의 “TotalCount 확인 후 전체 페이지”를 그대로 수행
   * - 실패 시 throw(=Fail-fast)
   */
  const fetchAllByClient = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

      // 1) initial: TotalCount 확인 (PageSize=1)
      const initialPayload: PayloadSteptaskSyouhin = {
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: clientName },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initial = await getSteptaskSyouhin(initialPayload) // 실패하면 throw
      const totalCount = initial.pagination.TotalCount

      if (typeof totalCount !== 'number') {
        throw new Error('TotalCount is invalid')
      }
      if (totalCount === 0) return []

      // 2) 全ページ取得（부분 성공 없음）
      const pageRequests: Promise<Awaited<ReturnType<typeof getSteptaskSyouhin>>>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        pageRequests.push(
          getSteptaskSyouhin({
            Offset: offset,
            PageSize: pageSize,
            View: initialPayload.View,
          })
        )
      }

      const pages = await Promise.all(pageRequests)
      const list = pages.flatMap(p => p.list)

      // 3) item 가공(원래 로직 그대로)
      await enrichItemsInPlace(list)

      return list
    },
    [getSteptaskSyouhin]
  )

  /** 외부에서 강제 재취득 */
  const forceRefresh = useCallback(
    async (clientName?: string) => {
      if (!clientName || clientName === '最寄り先を選択') return
      if (!Array.isArray(customers) || customers.length === 0) return

      setListLoading(true)
      try {
        const list = await fetchAllByClient(clientName)

        setTouenItems(list)
        setDataEvaluatedOnce(true)

        const products = await getInitialProductsForClient(
          clientName,
          customers,
          list,
          obj => setItemObject(obj)
        )
        setProducts(products)
        setProductsMap(prev => ({ ...prev, [clientName]: products }))

        markRan(clientName)
      } catch (e) {
        console.error(e)
        setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        setListLoading(false)
      }
    },
    [
      customers,
      fetchAllByClient,
      markRan,
      setItemObject,
      setProducts,
      setProductsMap,
      setTouenItems,
      showBoundary,
    ]
  )

  /**
   * nearestClientName 변화 시:
   * - previewRowsOnce(更新日時)와 ranForClientRef의 timestamp(=previewTs) 비교로 one-shot 동작
   * - 필요할 때만 fetch / 이미 최신이면 스킵
   */
  useEffect(() => {
    if (!nearestClientName || nearestClientName === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return

    const currentPreviewTs = getPreviewTsForClient(nearestClientName)
    const lastTs = ranForClientRef.current.get(nearestClientName) ?? -1

    // one-shot: 이미 최신이면 스킵
    if (oneShotPerClient && lastTs >= currentPreviewTs && currentPreviewTs > 0) {
      // 스킵하더라도, 아직 한 번도 평가되지 않았다면(초기 진입) 완료 상태로만 맞춘다.
      if (!dataEvaluatedOnce) setDataEvaluatedOnce(true)

      // products가 비어있는 상황이면, 현재 state(touenItems)로 초기 상품만 맞춰준다(네트워크 없음)
      ;(async () => {
        try {
          const source = touenItems ?? []
          if (source.length === 0) return

          const products = await getInitialProductsForClient(
            nearestClientName,
            customers,
            source,
            obj => setItemObject(obj)
          )
          setProducts(products)
          setProductsMap(prev => ({ ...prev, [nearestClientName]: products }))
        } catch {
          // noop
        }
      })()

      return
    }

    let aborted = false
    setListLoading(true)

    ;(async () => {
      try {
        const list = await fetchAllByClient(nearestClientName)
        if (aborted) return

        setTouenItems(list)
        setDataEvaluatedOnce(true)

        const products = await getInitialProductsForClient(
          nearestClientName,
          customers,
          list,
          obj => setItemObject(obj)
        )
        if (aborted) return

        setProducts(products)
        setProductsMap(prev => ({ ...prev, [nearestClientName]: products }))

        // one-shot 타임스탬프는 previewTs 기준으로 저장
        markRan(nearestClientName, currentPreviewTs || Date.now())
      } catch (e) {
        console.error(e)
        setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (!aborted) {
          setListLoading(false)
        }
      }
    })()

    return () => {
      aborted = true
      setListLoading(false)
    }
  }, [
    nearestClientName,
    customers,
    oneShotPerClient,
    fetchAllByClient,
    getPreviewTsForClient,
    markRan,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    showBoundary,
    dataEvaluatedOnce,
    touenItems,
  ])

  const fetchComplete = !listLoading && da

```
