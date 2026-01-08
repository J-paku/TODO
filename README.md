```
'use client'

import { useCallback, useEffect, useMemo, useRef, useState } from 'react'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'
import { getSteptaskSyouhin } from '../services/fetchSteptask' // 기존 사용 경로 유지
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}`
type ClassHash = Partial<Record<ClassKey, string>>
type PakuCustomHash = Partial<Record<`Custom${string | number}`, string>>
type DescriptionKey = `Description${Uppercase<string>}`
type DescriptionHash = Partial<Record<DescriptionKey, string>>

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

  /** 互換維持: index.ts から渡されるので受ける（本フック内では基本参照しない） */
  touenItems?: SteptaskItem[]
  setTouenItems: Dispatch<SetStateAction<SteptaskItem[]>>

  /** 互換維持: 外で使うので受ける（このフックでは触らない） */
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

/** nearestClientName が “未選択扱い” かどうか（あなたの既存仕様に合わせて調整可） */
const isInvalidClient = (name: string | null | undefined) => {
  if (!name) return true
  const n = String(name).trim()
  return n.length === 0 || n === '最寄り先を選択'
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()

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

  /**
   * client単位の制御:
   * - lastPreviewTs: “この preview の更新日時まで” 取得済み
   * - inFlight: 同一 client の並列二重呼び出しを dedupe
   */
  const clientStateRef = useRef<
    Map<
      string,
      {
        lastPreviewTs: number
        inFlight?: Promise<SteptaskItem[]>
      }
    >
  >(new Map())

  /** previewRowsOnce から 해당 client의 preview timestamp 얻기 */
  const getPreviewTsForClient = useCallback(
    (clientName: string): number | null => {
      const rows = previewRowsOnce ?? []
      const row = rows.find(r => {
        const title = r?.タイトル ?? r?.Title ?? ''
        return String(title).trim() === String(clientName).trim()
      })
      const t = ts(row?.更新日時 ?? row?.UpdatedTime ?? row?.updatedTime)

      // 중요: row 자체를 못 찾는 경우에는 “스킵 판단”을 하면 안 됨
      // -> null 로 반환해서 oneShot gate를 적용하지 않게 한다.
      const found = !!row
      return found ? t : null
    },
    [previewRowsOnce]
  )

  /** 핵심 fetch 함수 (중복호출 방지 + oneShot 게이트 + 에러 정책 통일) */
  const fetchClient = useCallback(
    async (clientName: string, opts?: { force?: boolean }) => {
      const force = opts?.force === true

      // oneShot 판단용 previewTs
      const previewTsOrNull = getPreviewTsForClient(clientName)

      const state = clientStateRef.current.get(clientName) ?? {
        lastPreviewTs: -1,
      }

      // 1) 진행중 요청이 있으면 재사용 (중복 호출 방지)
      if (!force && state.inFlight) {
        return state.inFlight
      }

      // 2) oneShot gate: previewTs를 알 수 있는 경우에만 적용
      if (!force && oneShotPerClient && previewTsOrNull !== null) {
        if (state.lastPreviewTs >= previewTsOrNull) {
          // 이미 최신 previewTs까지 반영됨
          return [] as SteptaskItem[] // 호출 스킵 (호출자에서 길이로 판단하지 않도록 주의)
        }
      }

      // 3) 실제 호출을 inFlight로 등록
      const p = (async () => {
        setListLoading(true)
        setDataEvaluatedOnce(false)
        const errorList: string[] = []
        try {
          const apiList: SteptaskItem[] = (await getSteptaskSyouhin(clientName)) ?? []
          setTouenItems(apiList)
          setDataEvaluatedOnce(true)

          // fetch 성공 시: lastPreviewTs 갱신
          const nextPreviewTs =
            previewTsOrNull !== null ? previewTsOrNull : Date.now() // previewTs 없으면 “현재시각”으로만 중복 방지
          clientStateRef.current.set(clientName, { lastPreviewTs: nextPreviewTs })

          return apiList
        } catch (e) {
          errorList.push(String(e))
          setDataEvaluatedOnce(true)
          console.error(e)
          // 기존 정책 유지: boundary로 에러 페이지 라우팅
          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
          throw e
        } finally {
          setListLoading(false)
          // inFlight 해제
          const cur = clientStateRef.current.get(clientName) ?? { lastPreviewTs: -1 }
          clientStateRef.current.set(clientName, { ...cur, inFlight: undefined })

          if (errorList.length > 0) {
            // 로그 정책 유지
            try {
              await logSteptaskErrorCause(errorList.join('\n'))
            } catch {
              // logging 실패는 무시(본 에러 흐름 방해 금지)
            }
          }
        }
      })()

      clientStateRef.current.set(clientName, { ...state, inFlight: p })
      return p
    },
    [getPreviewTsForClient, oneShotPerClient, setTouenItems, showBoundary]
  )

  /**
   * 외부에서 호출 가능한 forceRefresh:
   * - 인자로 clientName 주면 그 client를 강제 재조회
   * - 인자가 없으면 nearestClientName을 사용
   */
  const forceRefresh = useCallback(
    (clientName?: string) => {
      const target = clientName ?? nearestClientName ?? null
      if (isInvalidClient(target)) return
      void fetchClient(String(target), { force: true })
    },
    [nearestClientName, fetchClient]
  )

  /** fetchComplete 가 안정적으로 true 된 다음 프레임에서 stableFetchComplete true */
  useEffect(() => {
    let raf: number | null = null
    const fetchComplete = !listLoading && dataEvaluatedOnce
    if (fetchComplete) {
      raf = requestAnimationFrame(() => setStableFetchComplete(true))
    } else {
      setStableFetchComplete(false)
    }
    return () => {
      if (raf !== null) cancelAnimationFrame(raf)
    }
  }, [listLoading, dataEvaluatedOnce])

  /**
   * 자동 fetch (원래 로직 유지 + IIFE 방식)
   * - nearestClientName이 유효하고 customers가 준비된 경우에만
   * - oneShotPerClient 조건을 fetchClient에서 처리
   */
  useEffect(() => {
    let cancelled = false
    ;(async () => {
      if (cancelled) return
      if (isInvalidClient(nearestClientName)) return
      if (!Array.isArray(customers) || customers.length === 0) return

      try {
        // force가 아닌 일반 fetch (oneShot 적용)
        await fetchClient(String(nearestClientName))
      } catch {
        // fetchClient 내부에서 boundary 처리하므로 여기서 추가 처리 불필요
      }
    })()
    return () => {
      cancelled = true
    }
  }, [nearestClientName, customers.length, fetchClient])

  const fetchComplete = useMemo(() => !listLoading && dataEvaluatedOnce, [listLoading, dataEvaluatedOnce])

  return {
    listLoading,
    fetchComplete,
    stableFetchComplete,
    forceRefresh,
  }
}

  ```
