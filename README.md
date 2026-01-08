```
import { useEffect, useRef, useState, useCallback } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import type { Customer, Product } from '@/api/loadCustomerData'
import { logSteptaskErrorCause } from '@/api/fetchSteptask'
import { getSteptaskSyouhin } from '../services/fetchSteptask'
import { useRouter } from 'next/router'
import { ObjectResult, SteptaskItem } from '../types/types'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'
import { useErrorBoundary } from 'react-error-boundary'

/** ---- 型定義 ---- */
type ClassKey = `Class${Uppercase<string>}` // 'ClassA' ~ 'ClassZ'
type ClassHash = Partial<Record<ClassKey, string>>
type PakuCustomHash = Partial<Record<`Custom${string | number}`, string>>
type DescriptionKey = `Description${Uppercase<string>}` // 'DescriptionA' ~ 'DescriptionZ'
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
  /** 最寄り先名（未選択時は null） */
  nearestClientName: string | null
  /** 一度だけ取得されたプレビュー行 */
  previewRowsOnce: PreviewRow[] | null
  customers: Customer[]
  /** 互換維持のため: 本フック内では参照しない（外部状態のみ更新） */
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
/** 値→タイムスタンプ（不正値は 0） */
const ts = (v: unknown): number => {
  const t = Date.parse(String(v ?? ''))
  return Number.isFinite(t) ? t : 0
}

/** ---- Hook 本体 ---- */
export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary() // 404.tsxにリンクする（/ABC）

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

  /** 実行済みタイムスタンプを得意先単位で保持（APIでも一度きり制御したい場合用） */
  const ranForClientRef = useRef<Map<string, number>>(new Map())

  /** 再取得の強制（得意先単位 or 全体） */
  const forceRefresh = useCallback(async (clientName?: string) => {
    if (!clientName) return
    setListLoading(true)
    try {
      const apiList: SteptaskItem[] = (await getSteptaskSyouhin(clientName)) ?? []
      setTouenItems(apiList)
      setDataEvaluatedOnce(true)
    } catch (e) {
      showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      console.error(e)
      setDataEvaluatedOnce(true)
    } finally {
      setListLoading(false)
      ranForClientRef.current.set(clientName, Date.now())
    }
  }, [])

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

  /** 顧客がロードされたら全体のワンショット制御を解除 */
  useEffect(() => {
    if (customers.length > 0) {
      forceRefresh()
    }
  }, [customers.length, forceRefresh])

  const router = useRouter()

  /** いつでも API → 整合確認 → 画面状態更新（IndexedDBは使用しない） */
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

    setListLoading(true)
    let aborted = false

    const errorList: string[] = [] // エラーメッセージArray

    ;(async () => {
      try {
        const apiList: SteptaskItem[] = (await getSteptaskSyouhin(nearestClientName)) ?? []
        if (!aborted) {
          setTouenItems(apiList)
          setDataEvaluatedOnce(true)
        }
      } catch (error) {
        setDataEvaluatedOnce(true)
        errorList.push(String(error)) // エラー文字列積み重ねる
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (!aborted) {
          setListLoading(false)
          ranForClientRef.current.set(nearestClientName, currentPreviewTs)
          if (errorList.length > 0) {
            await logSteptaskErrorCause(errorList.join('\n'))
            showBoundary(
              new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR)
            )
          }
        }
      }
    })()

    return () => {
      aborted = true
      setListLoading(false)
    }
  }, [
    String(nearestClientName),
    previewRowsOnce,
    customers,
    oneShotPerClient,
    setItemObject,
    setTouenItems,
    setProducts,
    setProductsMap,
  ])

  const fetchComplete = !listLoading && dataEvaluatedOnce
  return { listLoading, fetchComplete, stableFetchComplete, forceRefresh }
}

```
