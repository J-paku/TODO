```
// api/touen/count/getSteptaskSyouhin.ts
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'

export type PayloadSteptaskSyouhin = {
  ApiVersion: number
  Offset: number
  PageSize: number
  View: {
    ColumnFilterHash: { Title: string }
    ColumnFilterSearchTypes: { Title: string }
  }
}

export type SteptaskItemsGetResponse<T = any> = {
  Response?: {
    TotalCount?: number
    Data?: T[]
  }
}

export default async function getSteptaskSyouhin<T = any>(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<SteptaskItemsGetResponse<T>>> {
  const response = await httpRequest(() =>
    axiosInstance.post(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return { code: response.code ?? 500, message: response.message, data: undefined }
  }

  return {
    code: 200,
    message: 'api get success',
    data: response?.data as SteptaskItemsGetResponse<T>,
  }
}

```
'use client'

import { useCallback, useEffect, useMemo, useRef, useState } from 'react'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import useTouenCount from '@/hooks/useTouenCount'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'

/**
 * useFetchTouenItems.ts
 * - index.ts에서 아래처럼 그대로 사용 가능:
 *   const { listLoading, stableFetchComplete, forceRefresh } = useFetchTouenItems({...})
 *
 * - 외부 getSteptaskSyouhin(service)은 사용하지 않고,
 *   useTouenCount().getSteptaskSyouhin(payload)만 호출해서
 *   TotalCount 기반으로 전 페이지를 가져온 뒤 setTouenItems로 반영합니다.
 *
 * 주의:
 * - 당신 프로젝트의 ApiResponse / Steptask 응답 형태가 약간 달라도 동작하도록 “유연 파싱”을 넣었습니다.
 * - item의 세부 가공(fetchDetail 등)은 여기 파일만으로는 정의가 없어서,
 *   최소한 “Hash 객체 보장”까지만 수행합니다(컴파일/실행 우선).
 */

/** ---- 최소 타입(프로젝트 타입이 있어도 충돌 최소화 위해 좁게 정의) ---- */
type AnyObj = Record<string, any>

export type ClassKey = `Class${Uppercase<string>}`
export type ClassHash = Partial<Record<ClassKey, string>>
export type PakuCustomHash = Partial<Record<`Custom${string | number}`, any>>
export type DescriptionKey = `Description${Uppercase<string>}`
export type DescriptionHash = Partial<Record<DescriptionKey, string>>

export interface SteptaskItem extends AnyObj {
  Title?: string
  タイトル?: string
  UpdatedTime?: string
  updatedTime?: string
  ResultID?: string
  ResultId?: string
  SiteId?: string
  ClassHash?: ClassHash & AnyObj

  // 아래는 기존 로직이 채우던 필드들(없어도 되지만 런타임 안정 위해 존재만 보장)
  PakuCustomHash?: PakuCustomHash
  PakuCustomHashTwo?: AnyObj
  PakuCustomHashThree?: AnyObj
  PakuCustomHashFour?: AnyObj
  PakuCustomSoko?: AnyObj
  PakuCustomHashProductIndex?: AnyObj
  PakuCustomHashMasterIndex?: AnyObj
}

export type ObjectResult = AnyObj
export type ProductsMap = Record<string, any[]>

export interface PreviewRow {
  タイトル?: string
  Title?: string
  更新日時?: string
  UpdatedTime?: string
  updatedTime?: string
}

export type UseTouenItemsParams = {
  /** 最寄り先名（未選択時は null） */
  nearestClientName: string | null
  /** 一度だけ取得されたプレビュー行 */
  previewRowsOnce: PreviewRow[] | null
  customers: any[]
  /** 互換維持のため: 本フック内では参照しない（外部状態のみ更新） */
  touenItems?: SteptaskItem[]
  setTouenItems: React.Dispatch<React.SetStateAction<SteptaskItem[]>>
  setItemObject: React.Dispatch<React.SetStateAction<ObjectResult>>
  setProducts: React.Dispatch<React.SetStateAction<any[]>>
  setProductsMap: React.Dispatch<React.SetStateAction<ProductsMap>>
  oneShotPerClient?: boolean
}

export type UseTouenItemsReturn = {
  listLoading: boolean
  fetchComplete: boolean
  stableFetchComplete: boolean
  forceRefresh: (clientName?: string) => void
}

/** ---- 유틸 ---- */
const parseTs = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

const isInvalidClient = (name: string | null | undefined) => {
  if (!name) return true
  const n = String(name).trim()
  return n.length === 0 || n === '最寄り先を選択'
}

/**
 * ApiResponse / raw 등을 최대한 유연하게 파싱해서
 * { totalCount, data } 형태로 통일합니다.
 */
const normalizeSteptaskResponse = (
  maybe: any
): { ok: boolean; totalCount?: number; data: SteptaskItem[] } => {
  if (!maybe) return { ok: false, data: [] }

  // 1) ApiResponse 형태: { code, data, message }
  const code = typeof maybe.code === 'number' ? maybe.code : undefined
  const body = maybe.data ?? maybe

  // 2) body 형태: { Response: { TotalCount, Data } }
  const resp = body?.Response ?? body?.response ?? body
  const totalCount =
    typeof resp?.TotalCount === 'number'
      ? resp.TotalCount
      : typeof resp?.totalCount === 'number'
        ? resp.totalCount
        : undefined

  const dataRaw = resp?.Data ?? resp?.data ?? []
  const data = Array.isArray(dataRaw) ? (dataRaw as SteptaskItem[]) : []

  const ok = code ? code === 200 : true
  return { ok, totalCount, data }
}

/** item의 Hash 객체들을 항상 존재하게 만들어 런타임 안정성 확보 */
const ensureItemHashes = (item: SteptaskItem) => {
  item.PakuCustomHash ||= {}
  item.PakuCustomHashTwo ||= {}
  item.PakuCustomHashThree ||= {}
  item.PakuCustomHashFour ||= {}
  item.PakuCustomSoko ||= {}
  item.ClassHash ||= {}
  item.PakuCustomHashProductIndex ||= {}
  item.PakuCustomHashMasterIndex ||= {}
  return item
}

/** ---- Hook 본체 ---- */
export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()
  const { getSteptaskSyouhin } = useTouenCount()

  const {
    nearestClientName,
    previewRowsOnce,
    customers,
    setTouenItems,
    oneShotPerClient = true,
  } = params

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  /** 得意先ごとの実行済みタイムスタンプ（ワンショット制御） */
  const ranForClientRef = useRef<Map<string, number>>(new Map())
  /** 同一得意先の同時実行をデデュープ */
  const inFlightRef = useRef<Map<string, Promise<SteptaskItem[]>>>(new Map())

  const previewTsForClient = useCallback(
    (clientName: string): number | null => {
      const rows = previewRowsOnce ?? []
      const row = rows.find(r => {
        const title = r?.タイトル ?? r?.Title ?? ''
        return String(title).trim() === String(clientName).trim()
      })
      if (!row) return null
      return parseTs(row.更新日時 ?? row.UpdatedTime ?? row.updatedTime)
    },
    [previewRowsOnce]
  )

  /**
   * TotalCount 기반 전 페이지 수집
   * - 훅 밖 URL/axios 관리: useTouenCount().getSteptaskSyouhin(payload) 호출만 수행
   */
  const fetchAllPages = useCallback(
    async (tokuisaki: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

      // 1) TotalCount 확인 (PageSize=1)
      const initialPayload = {
        ApiVersion: 1.1,
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: tokuisaki },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initialRes = await getSteptaskSyouhin(initialPayload as any)
      const initialNorm = normalizeSteptaskResponse(initialRes)

      if (!initialNorm.ok) throw new Error('getSteptaskSyouhin initial failed')
      if (typeof initialNorm.totalCount !== 'number') throw new Error('TotalCountが不正です')

      const totalCount = initialNorm.totalCount

      // 2) 전 페이지 병렬 호출
      const reqs: Promise<any>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        const payload = { ...initialPayload, Offset: offset, PageSize: pageSize }
        reqs.push(getSteptaskSyouhin(payload as any))
      }

      const pages = await Promise.all(reqs)
      const list: SteptaskItem[] = pages.flatMap(p => normalizeSteptaskResponse(p).data)

      // 3) 최소 가공(해시 보장)
      for (const item of list) ensureItemHashes(item)

      return list
    },
    [getSteptaskSyouhin]
  )

  const runFetchForClient = useCallback(
    async (clientName: string, opts?: { force?: boolean }): Promise<SteptaskItem[]> => {
      const force = opts?.force === true

      // in-flight dedupe
      if (!force) {
        const inflight = inFlightRef.current.get(clientName)
        if (inflight) return inflight
      }

      const previewTs = previewTsForClient(clientName)
      const lastTs = ranForClientRef.current.get(clientName) ?? -1

      // oneShotPerClient: previewTs가 있을 때만 비교(없으면 항상 실행)
      if (!force && oneShotPerClient && previewTs !== null && lastTs >= previewTs) {
        // 이미 최신으로 판단 -> 아무것도 안 바꾸고 종료
        setDataEvaluatedOnce(true)
        return []
      }

      setListLoading(true)
      setDataEvaluatedOnce(false)

      const errorList: string[] = []

      const p = (async () => {
        try {
          const list = await fetchAllPages(clientName)
          setTouenItems(list)
          setDataEvaluatedOnce(true)

          const nextTs = previewTs ?? Date.now()
          ranForClientRef.current.set(clientName, nextTs)

          return list
        } catch (e) {
          errorList.push(String(e))
          setDataEvaluatedOnce(true)
          console.error(e)
          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
          throw e
        } finally {
          setListLoading(false)
          inFlightRef.current.delete(clientName)

          if (errorList.length > 0) {
            try {
              await logSteptaskErrorCause(errorList.join('\n'))
            } catch {
              // noop
            }
          }
        }
      })()

      inFlightRef.current.set(clientName, p)
      return p
    },
    [fetchAllPages, oneShotPerClient, previewTsForClient, setTouenItems, showBoundary]
  )

  /** 외부에서 강제 재조회 */
  const forceRefresh = useCallback(
    (clientName?: string) => {
      const target = clientName ?? nearestClientName
      if (isInvalidClient(target)) return
      void runFetchForClient(String(target), { force: true })
    },
    [nearestClientName, runFetchForClient]
  )

  /** stableFetchComplete: fetchComplete가 프레임 경계에서 안정화 되었는지 */
  useEffect(() => {
    let rafId: number | null = null
    const fetchComplete = !listLoading && dataEvaluatedOnce
    if (fetchComplete) {
      rafId = requestAnimationFrame(() => setStableFetchComplete(true))
    } else {
      setStableFetchComplete(false)
    }
    return () => {
      if (rafId !== null) cancelAnimationFrame(rafId)
    }
  }, [listLoading, dataEvaluatedOnce])

  /**
   * IIFE 방식(B)
   * - nearestClientName / customers 준비되면 자동 fetch
   */
  useEffect(() => {
    let cancelled = false
    ;(async () => {
      if (cancelled) return
      if (isInvalidClient(nearestClientName)) return
      if (!Array.isArray(customers) || customers.length === 0) return

      try {
        await runFetchForClient(String(nearestClientName))
      } catch {
        // 에러 처리는 showBoundary에서 수행
      }
    })()

    return () => {
      cancelled = true
    }
  }, [nearestClientName, customers?.length, runFetchForClient])

  const fetchComplete = useMemo(() => !listLoading && dataEvaluatedOnce, [listLoading, dataEvaluatedOnce])

  return {
    listLoading,
    fetchComplete,
    stableFetchComplete,
    forceRefresh,
  }
}

```
