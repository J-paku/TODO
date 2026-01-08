```
import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'
import { getSteptaskSyouhin } from '../services/fetchSteptask'
import { ObjectResult, SteptaskItem } from '../types/types'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'

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

/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

/** 正規化 */
const normName = (v: unknown) => String(v ?? '').trim()

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()

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

  /** 得意先ごとに「最後に適用したプレビューTS」を保持（one-shot判定用） */
  const appliedTsRef = useRef<Map<string, number>>(new Map())

  /** 同一得意先の同時リクエストは 1本にまとめる（StrictModeの二重effectも吸収） */
  const inflightRef = useRef<Map<string, Promise<void>>>(new Map())

  /** 最新リクエストのみ反映（遅延レスポンスの上書き防止） */
  const reqIdRef = useRef(0)

  /** unmount後 setState 防止 */
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

  /**
   * 実際の取得処理（API呼び出しはここだけ）
   * - in-flight統合
   * - 最新reqのみ反映
   * - one-shot判定（caller側で事前判定し、ここでは必要に応じてforceで無視）
   */
  const fetchClient = useCallback(
    async (clientName: string, opts?: { previewTs?: number; force?: boolean }) => {
      const name = normName(clientName)
      if (!name || name === '最寄り先を選択') return
      if (!Array.isArray(customers) || customers.length === 0) return

      const previewTs = opts?.previewTs ?? 0
      const force = Boolean(opts?.force)

      // one-shot: 既に同じ/新しいTSを適用済みならスキップ（forceは無視）
      if (!force && oneShotPerClient && previewTs > 0) {
        const applied = appliedTsRef.current.get(name)
        if (applied !== undefined && applied >= previewTs) return
      }

      // in-flight: 同一clientの連打/二重effectは1本にまとめる
      const inflight = inflightRef.current.get(name)
      if (inflight) return inflight

      const myReqId = ++reqIdRef.current
      setListLoading(true)

      const run = (async () => {
        const errorList: string[] = []
        try {
          const apiList: SteptaskItem[] = (await getSteptaskSyouhin(name)) ?? []

          // unmount or newer request happened
          if (!aliveRef.current) return
          if (myReqId !== reqIdRef.current) return

          setTouenItems(apiList)
          setDataEvaluatedOnce(true)

          // 取得成功時に適用TSを更新（previewTsが無い場合は“実行時刻”）
          appliedTsRef.current.set(name, previewTs > 0 ? previewTs : Date.now())
        } catch (e) {
          errorList.push(String(e))
          if (!aliveRef.current) return
          setDataEvaluatedOnce(true)

          // ログ送信（失敗しても本筋は継続）
          try {
            await logSteptaskErrorCause(errorList.join('\n'))
          } catch (_) {}

          showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
        } finally {
          inflightRef.current.delete(name)
          if (!aliveRef.current) return
          // 最新reqのみローディングを落とす
          if (myReqId === reqIdRef.current) setListLoading(false)
        }
      })()

      inflightRef.current.set(name, run)
      return run
    },
    [
      customers,
      oneShotPerClient,
      setTouenItems,
      showBoundary,
      // 互換維持のため依存に残す（外部がこれらに依存している想定）
      setItemObject,
      setProducts,
      setProductsMap,
    ]
  )

  /** 手動再取得（元のAPIと同じ名前で提供） */
  const forceRefresh = useCallback(
    (clientName?: string) => {
      const name = normName(clientName ?? nearestClientName)
      if (!name || name === '最寄り先を選択') return
      // force: one-shot判定を無視して必ず再取得
      void fetchClient(name, { force: true, previewTs: 0 })
    },
    [nearestClientName, fetchClient]
  )

  /**
   * 自動取得（元の挙動維持）
   * - nearestClientName が有効
   * - customers がロード済み
   * - previewRowsOnce から該当得意先の更新TSを取得し one-shot 判定
   */
  useEffect(() => {
    const name = normName(nearestClientName)
    if (!name || name === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return

    const rows: PreviewRow[] = previewRowsOnce ?? []
    const rowForClient = rows.find(r => {
      const title = normName(r?.タイトル ?? r?.Title ?? '')
      return title === name
    })
    const previewTs = ts(rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime)

    // previewTsが取れない場合でも「選択されたら取得」は維持（= previewTs=0扱い）
    void fetchClient(name, { previewTs, force: false })
  }, [nearestClientName, previewRowsOnce, customers, fetchClient])

  /**
   * ここが元コードの非効率ポイントだったので修正：
   * customersロード時に forceRefresh() を“引数なし”で呼んでも意味がない。
   * → nearestClientNameが既に選択済みなら、同じ自動取得effectが必ず走るので不要。
   * よってこの useEffect 自体を削除するのが最も安全で効率的。
   */

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
