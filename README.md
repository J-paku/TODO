```
import { useCallback, useEffect, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import { getSteptaskSyouhin } from '../services/fetchSteptask'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'
import { useErrorBoundary } from 'react-error-boundary'
import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

type ProductsMap = Record<string, Product[]>

type UseTouenItemsParams = {
  setTouenItems: Dispatch<SetStateAction<SteptaskItem[]>>
  setItemObject: Dispatch<SetStateAction<ObjectResult>>
  setProducts: Dispatch<SetStateAction<Product[]>>
  setProductsMap: Dispatch<SetStateAction<ProductsMap>>
}

type UseTouenItemsReturn = {
  listLoading: boolean
  fetchComplete: boolean
  stableFetchComplete: boolean
  /** 明示的に呼ぶ：これ以外では取得しない */
  fetchTouenItems: (clientName: string, options?: { versionTs?: number; force?: boolean }) => Promise<void>
  /** 任意：直近取得したclient */
  lastClientName: string | null
}

/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()

  const { setTouenItems, setItemObject, setProducts, setProductsMap } = params

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)
  const [lastClientName, setLastClientName] = useState<string | null>(null)

  /** 最新リクエストのみ反映するためのID */
  const reqIdRef = useRef(0)
  /** clientごとの最新反映済みversionTs（任意でスキップに使う） */
  const appliedVersionRef = useRef<Map<string, number>>(new Map())
  /** in-flight（同一clientの連打）を1本にまとめる */
  const inflightRef = useRef<Map<string, Promise<void>>>(new Map())
  /** unmount後のsetState防止 */
  const aliveRef = useRef(true)

  useEffect(() => {
    aliveRef.current = true
    return () => {
      aliveRef.current = false
    }
  }, [])

  /** fetchComplete がフレーム境界で安定したことを別フラグへ反映 */
  useEffect(() => {
    let rafId: number | null = null
    if (!listLoading && dataEvaluatedOnce) {
      rafId = requestAnimationFrame(() => {
        if (aliveRef.current) setStableFetchComplete(true)
      })
    } else {
      setStableFetchComplete(false)
    }
    return () => {
      if (rafId !== null) cancelAnimationFrame(rafId)
    }
  }, [listLoading, dataEvaluatedOnce])

  const fetchTouenItems = useCallback(
    async (clientName: string, options?: { versionTs?: number; force?: boolean }) => {
      const name = String(clientName ?? '').trim()
      if (!name) return

      const versionTs = ts(options?.versionTs)
      const force = Boolean(options?.force)

      // 既に同じversionを適用済みならスキップ（forceなら無視）
      if (!force) {
        const applied = appliedVersionRef.current.get(name)
        if (applied !== undefined && applied >= versionTs && versionTs !== 0) {
          return
        }
      }

      // 同一clientの連打は in-flight を再利用
      const existing = inflightRef.current.get(name)
      if (existing) return existing

      const myReqId = ++reqIdRef.current
      setListLoading(true)
      setLastClientName(name)

      const run = (async () => {
        const errorList: string[] = []
        try {
          const apiList: SteptaskItem[] = (await getSteptaskSyouhin(name)) ?? []

          // 途中で別リクエストが走っていたら反映しない（最新のみ）
          if (!aliveRef.current) return
          if (myReqId !== reqIdRef.current) return

          setTouenItems(apiList)
          setDataEvaluatedOnce(true)

          // versionTs が 0 の場合は「取得時刻」を入れておく（任意）
          appliedVersionRef.current.set(name, versionTs !== 0 ? versionTs : Date.now())
        } catch (e) {
          errorList.push(String(e))
          if (!aliveRef.current) return
          setDataEvaluatedOnce(true)
          // 必要ならログ送信
          try {
            await logSteptaskErrorCause(errorList.join('\n'))
          } catch (_) {
            // ログ失敗は握りつぶし（取得失敗の本筋ではない）
          }
          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
        } finally {
          if (!aliveRef.current) return
          // 最新リクエストだけがローディングを落とす
          if (myReqId === reqIdRef.current) setListLoading(false)
          inflightRef.current.delete(name)
        }
      })()

      inflightRef.current.set(name, run)
      return run
    },
    [setTouenItems, showBoundary]
  )

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, fetchTouenItems, lastClientName }
}

```
