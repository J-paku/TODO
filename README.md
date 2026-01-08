```
'use client'

import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'
import useTouenCount from '@/hooks/useTouenCountActions' // 프로젝트에서 실제 경로에 맞게 유지/조정
import type {
  PayloadSteptaskSyouhin,
  StepTaskItemsGetResponse,
} from '@/api/touen/count/getSteptaskSyouhin'

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}` // 'ClassA' ~
type ClassHash = Partial<Record<ClassKey, string>>
type PakuCustomHash = Partial<Record<`Custom${string | number}`, string>>
type DescriptionKey = `Description${Uppercase<string>}` // 'DescriptionA' ~
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
  // StepTaskの他フィールドがあっても問題ない（unknown拡張）
  [key: string]: unknown
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
  touenItems?: TouenItem[]
  setTouenItems: Dispatch<SetStateAction<TouenItem[]>>
  setItemObject: Dispatch<SetStateAction<Record<string, unknown>>>
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

/**
 * StepTask ITEMS_GET を「TotalCount→全件ページング」して配列で返す
 * - api/touen/count/getSteptaskSyouhin.ts は Response{TotalCount,Data} を保持して返す前提
 */
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
    // 下記は互換維持: 現状このフック内では参照しない
    setItemObject,
    setProducts,
    setProductsMap,
    oneShotPerClient = true,
  } = params

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  /**
   * 実行済みタイムスタンプを得意先単位で保持
   * - 「プレビュー更新日時 <= 最後に取得した日時」なら再取得しない
   */
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
   * customers がロードされたら「現状の nearestClientName が有効なら」その得意先で一度取得
   * - 以前の実装は forceRefresh() が何もせず終了していたため「何も取れない」状態になりやすかった
   */
  useEffect(() => {
    if (customers.length <= 0) return
    if (!isValidClientName(nearestClientName)) return
    // customersが揃ったタイミングで、現得意先を強制取得（旧挙動に近い）
    forceRefresh(nearestClientName)
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [customers.length])

  /**
   * 実データ取得（TotalCount→全件）
   * - ここを「元の getSteptaskSyouhin の取得部分」をフック側に寄せた形
   */
  const fetchAllPages = useCallback(
    async (clientName: string): Promise<TouenItem[]> => {
      const pageSize = 200

      // 1) TotalCount 取得
      const initialPayload = buildInitialPayload(clientName)
      const initialRes: StepTaskItemsGetResponse<TouenItem> | undefined =
        await getSteptaskSyouhin<TouenItem>(initialPayload)

      const totalCount = initialRes?.Response?.TotalCount
      if (typeof totalCount !== 'number') {
        throw new Error('StepTask TotalCount が不正です')
      }

      // 2) ページング取得
      const requests: Promise<StepTaskItemsGetResponse<TouenItem> | undefined>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        const payload: PayloadSteptaskSyouhin = {
          ...initialPayload,
          Offset: offset,
          PageSize: pageSize,
        }
        requests.push(getSteptaskSyouhin<TouenItem>(payload))
      }

      const responses = await Promise.all(requests)
      const list: TouenItem[] = responses.flatMap(r => r?.Response?.Data ?? [])

      // 3) ここで item 加工が必要ならこの位置に入れる（現時点では「取得ロジック移植」に集中）
      //    - 以前の巨大 getSteptaskSyouhin 内の per-item 加工をここへ移植する場合もここが正しい場所

      return list
    },
    [getSteptaskSyouhin]
  )

  /**
   * 再取得の強制（得意先単位 or 現得意先）
   * - clientName 未指定の場合は nearestClientName を使う（「何も取れない」回避）
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
        // 強制取得は「今」基準で stamp
        ranForClientRef.current.set(target, Date.now())
      }
    },
    [fetchAllPages, nearestClientName, setTouenItems, showBoundary]
  )

  /**
   * nearestClientName / previewRowsOnce 変化での自動取得（ワンショット制御あり）
   * - “元の動き” を維持するため、プレビュー更新日時を見て必要時だけ取り直す
   */
  useEffect(() => {
    if (!isValidClientName(nearestClientName)) return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows: PreviewRow[] = previewRowsOnce ?? []
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
    setItemObject,
    setProducts,
    setProductsMap,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
