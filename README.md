```
'use client'

import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'
import useTouenCount from './useTouenCountActions' // 실제 경로에 맞게 유지/조정
import type { ObjectResult, SteptaskItem } from '../types/types'

/** ---- 型定義 ---- */
type ProductsMap = Record<string, Product[]>

interface PreviewRow {
  タイトル?: string
  Title?: string
  更新日時?: string
  UpdatedTime?: string
  updatedTime?: string
}

type UseTouenItemsParams = {
  /** 最寄り先名（未選択時は null） */
  nearestClientName: string | null
  /** 一度だけ取得されたプレビュー行 */
  previewRowsOnce: PreviewRow[] | null
  customers: Customer[]

  /** index.tsx 互換維持 */
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
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

const isValidClientName = (v: unknown): v is string => {
  const s = String(v ?? '').trim()
  return s.length > 0 && s !== '最寄り先を選択'
}

const getPreviewTsForClient = (rows: PreviewRow[], clientName: string): number => {
  const rowForClient = rows.find(r => {
    const title = String(r?.タイトル ?? r?.Title ?? '').trim()
    return title === String(clientName).trim()
  })
  return ts(rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime)
}

/** StepTask ITEMS_GET payload (당신 프로젝트의 실제 payload 형태로 맞춰서 사용) */
type PayloadSteptaskSyouhin = {
  ApiVersion: number
  Offset: number
  PageSize: number
  View: {
    ColumnFilterHash: { Title: string }
    ColumnFilterSearchTypes: { Title: string }
  }
}

/** StepTask response 최소 형태(타입 충돌 방지용) */
type StepTaskItemsGetResponse<T> = {
  Response?: {
    TotalCount?: number
    Data?: T[]
  }
}

const buildInitialPayload = (clientName: string): PayloadSteptaskSyouhin => ({
  ApiVersion: 1.1,
  Offset: 0,
  PageSize: 1,
  View: {
    ColumnFilterHash: { Title: clientName },
    ColumnFilterSearchTypes: { Title: 'ExactMatch' },
  },
})

/** ---- Hook 本体 ---- */
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
    if (isValidClientName(nearestClientName)) {
      ranForClientRef.current.delete(nearestClientName)
    }
  }, [nearestClientName])

  /**
   * StepTask ITEMS_GET を「TotalCount→全件ページング」して配列で返す
   * 注意: useTouenCount().getSteptaskSyouhin は “Response(TotalCount/Data)” を返す前提
   */
  const fetchAllPages = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

      const initialPayload = buildInitialPayload(clientName)

      // 1) TotalCount
      const initialResUnknown = await getSteptaskSyouhin(initialPayload as any)
      const initialRes = initialResUnknown as StepTaskItemsGetResponse<SteptaskItem> | undefined

      const totalCount = initialRes?.Response?.TotalCount
      if (typeof totalCount !== 'number') {
        throw new Error('StepTask TotalCount が不正です')
      }

      // 2) 全ページ
      const reqs: Promise<unknown>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        const payload: PayloadSteptaskSyouhin = {
          ...initialPayload,
          Offset: offset,
          PageSize: pageSize,
        }
        reqs.push(getSteptaskSyouhin(payload as any))
      }

      const resList = await Promise.all(reqs)
      const list = resList.flatMap(r => {
        const rr = r as StepTaskItemsGetResponse<SteptaskItem> | undefined
        return rr?.Response?.Data ?? []
      })

      return list
    },
    [getSteptaskSyouhin]
  )

  /**
   * 強制再取得
   * - clientName 未指定なら nearestClientName を使う（「何も取れない」対策）
   */
  const forceRefresh = useCallback(
    async (clientName?: string) => {
      const target = isValidClientName(clientName)
        ? clientName
        : isValidClientName(nearestClientName)
          ? nearestClientName
          : null

      if (!target) return

      setListLoading(true)
      try {
        const list = await fetchAllPages(target)
        setTouenItems(list)
        setDataEvaluatedOnce(true)
      } catch (e) {
        console.error(e)
        setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        setListLoading(false)
        ranForClientRef.current.set(target, Date.now())
      }
    },
    [fetchAllPages, nearestClientName, setTouenItems, showBoundary]
  )

  /**
   * customers 로드 이후, 현재 nearestClientName 이 유효하면 한번 강제 로드
   * (이전 코드에서 forceRefresh()가 “아무것도 안 하는” 버그였던 부분 해결)
   */
  useEffect(() => {
    if (customers.length <= 0) return
    if (!isValidClientName(nearestClientName)) return
    forceRefresh(nearestClientName)
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [customers.length])

  /**
   * nearestClientName / previewRowsOnce 변화에 따른 자동 로드 (oneShot 제어)
   */
  useEffect(() => {
    if (!isValidClientName(nearestClientName)) return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows = previewRowsOnce ?? []
    const currentPreviewTs = getPreviewTsForClient(rows, nearestClientName)

    const lastTs = ranForClientRef.current.get(nearestClientName) ?? -1
    if (oneShotPerClient && lastTs >= currentPreviewTs) return

    setListLoading(true)
    let aborted = false

    ;(async () => {
      try {
        const list = await fetchAllPages(nearestClientName)
        if (!aborted) {
          setTouenItems(list)
          setDataEvaluatedOnce(true)
        }
      } catch (e) {
        console.error(e)
        if (!aborted) setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (!aborted) {
          setListLoading(false)
          ranForClientRef.current.set(nearestClientName, currentPreviewTs)
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
    fetchAllPages,
    setTouenItems,
    showBoundary,
    // 互換のため依存関係に残す（index側のstateセットと整合）
    setItemObject,
    setProducts,
    setProductsMap,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
