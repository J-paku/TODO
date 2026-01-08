```
'use client'

import { useCallback, useEffect, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'

// サービスレイヤー（既存構造を維持）
import useTouenCount from './useTouenCountActions'
import type { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

// 既存型
import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

// 既存ロジック（商品初期化）
import { getInitialProductsForClient } from '../lib/getInitialProductsForClient'

// 元ロジック依存（パスは既存のまま）
import { alphabetClass, numberClass } from '../services/classesConstants'
import { fetchDetail } from '../services/steptaskDetailService'
import { normalizePrice } from '../services/format'

/** ---- 型定義 ---- */
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

/**
 * StepTask item 加工（元の getSteptaskSyouhin 内の加工をそのまま移管）
 * - 元ロジック通り mutate する
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

    const classHash = item.ClassHash as Record<string, string>

    // 値ありキー抽出
    const alphaFirstKeys = alphabetClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })
    const alphaSecondKeys = numberClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })

    // 末尾値ありインデックス
    const alphabetMasterIndex =
      alphabetClass
        .map((k, idx) => ({ idx, v: classHash?.[k] }))
        .filter(({ v }) => String(v ?? '').trim() !== '')
        .at(-1)?.idx ?? -1

    const numberMasterIndex =
      numberClass
        .map((k, idx) => ({ idx, v: classHash?.[k] }))
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
      const value = classHash[key]
      if (!value) return
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

    // 数字側（Custom001〜）
    alphaSecondKeys.forEach((key, idx) => {
      const value = classHash[key]
      if (!value) return
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

    // MasterIndex付番
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
      const value = classHash[key]
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
      const value = classHash[key]
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

    // 単一コミット
    Object.assign(item.PakuCustomHash as Record<string, string | null>, draftPakuCustomHash)
    Object.assign(item.PakuCustomHashTwo as Record<string, string>, draftPakuCustomHashTwo)
    Object.assign(item.PakuCustomHashThree as Record<string, string>, draftPakuCustomHashThree)
    Object.assign(item.PakuCustomHashFour as Record<string, string>, draftPakuCustomHashFour)

    // 注意：元ロジックは ClassHash を Custom キーで上書きする
    Object.assign(item.ClassHash as Record<string, string>, draftClassHash)

    Object.assign(item.PakuCustomHashProductIndex as Record<string, number>, draftPakuCustomHashProductIndex)
    Object.assign(item.PakuCustomHashMasterIndex as Record<string, number>, draftPakuCustomHashMasterIndex)
  }
}

export function useFetchTouenItems(params: UseTouenItemsParams): UseTouenItemsReturn {
  const { showBoundary } = useErrorBoundary()

  const {
    nearestClientName,
    customers,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
  } = params

  const { getSteptaskSyouhin } = useTouenCount()

  const [listLoading, setListLoading] = useState(false)
  const [dataEvaluatedOnce, setDataEvaluatedOnce] = useState(false)
  const [stableFetchComplete, setStableFetchComplete] = useState(false)

  // 同一得意先への多重 fetch を防止
  const inFlightRef = useRef<Set<string>>(new Set())
  // 同じ依存関係で effect が再評価されても一度だけにするためのキー
  const lastRequestedKeyRef = useRef<string>('')

  // fetchComplete がフレーム境界で安定したことを別フラグへ反映
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

  /**
   * ページ全取得（Title ExactMatch）
   * - API は「ページ raw」を返す想定
   * - TotalCount を initial で取得し、全ページを並列取得する
   */
  const fetchAllByClient = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

      // 初回：TotalCount 確認（PageSize=1）
      const initialPayload: PayloadSteptaskSyouhin = {
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: clientName },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initial = await getSteptaskSyouhin(initialPayload)
      const totalCount = initial?.pagination?.TotalCount ?? 0
      if (totalCount <= 0) return []

      // 全ページ取得
      const pageRequests: Promise<ReturnType<typeof getSteptaskSyouhin>>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        pageRequests.push(
          getSteptaskSyouhin({
            Offset: offset,
            PageSize: pageSize,
            View: initialPayload.View,
          })
        )
      }

      const responses = await Promise.all(pageRequests)
      const list = responses.flatMap(r => r?.list ?? [])

      // item 加工（元ロジック）
      await enrichItemsInPlace(list)

      return list
    },
    [getSteptaskSyouhin]
  )

  /**
   * 外部からの強制更新
   * - index.tsx を変えない前提でも、将来的に利用できるよう維持する
   */
  const forceRefresh = useCallback(
    async (clientName?: string) => {
      if (!clientName || clientName === '最寄り先を選択') return
      if (inFlightRef.current.has(clientName)) return

      inFlightRef.current.add(clientName)
      setListLoading(true)

      try {
        const list = await fetchAllByClient(clientName)

        setTouenItems(list)
        setDataEvaluatedOnce(true)

        const products = await getInitialProductsForClient(
          clientName,
          customers,
          list,
          obj => setItemObject(obj),
          list
        )

        setProducts(products)
        setProductsMap(prev => ({ ...prev, [clientName]: products }))
      } catch (e) {
        console.error(e)
        setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        setListLoading(false)
        inFlightRef.current.delete(clientName)
      }
    },
    [customers, fetchAllByClient, setItemObject, setProducts, setProductsMap, setTouenItems, showBoundary]
  )

  /**
   * 自動 fetch
   * - nearestClientName が変わったら必ず最新取得
   * - previewRowsOnce などの one-shot 判定は行わない
   * - 無限ループ防止のため、同一キーでの再入を lastRequestedKey で抑止する
   */
  useEffect(() => {
    if (!nearestClientName || nearestClientName === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return

    const key = `${String(nearestClientName).trim()}::${customers.length}`
    if (lastRequestedKeyRef.current === key) return
    lastRequestedKeyRef.current = key

    if (inFlightRef.current.has(nearestClientName)) return

    let aborted = false
    inFlightRef.current.add(nearestClientName)
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
          obj => setItemObject(obj),
          list
        )
        if (aborted) return

        setProducts(products)
        setProductsMap(prev => ({ ...prev, [nearestClientName]: products }))
      } catch (e) {
        console.error(e)
        setDataEvaluatedOnce(true)
        showBoundary(new Error(TOUEN_NEW_ERROR_MESSAGES.TRANSACTION_SYOUHINBETSU_MASTER_GET_ERROR))
      } finally {
        if (!aborted) {
          setListLoading(false)
        }
        inFlightRef.current.delete(nearestClientName)
      }
    })()

    return () => {
      aborted = true
      setListLoading(false)
      inFlightRef.current.delete(nearestClientName)
    }
  }, [
    nearestClientName,
    customers,
    fetchAllByClient,
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
