```
// React
import { useEffect, useState, useCallback } from 'react'
// Hooks
import { useTouenInitializer } from './hooks/useTouenInitializer'
import { usePrintTouenOrder } from './hooks/usePrintTouenOrder'
import { useManageTouenPopup } from './hooks/useManageTouenPopup'
import { useNowTime } from './hooks/useNowTime'
import { useLockBodyScroll } from './hooks/useLockBodyScroll'
import { useFetchTouenItems } from './hooks/useFetchTouenItems'
// Libs
import { createIsIncrement } from './lib/pressHandlers'
// Components
import { hapticOn } from 'components/hapticOn'
import ClientSearchModal from 'components/ClientSearchModal'
import ClientSelectorBox from 'components/ClientSelectorBox'
import ToastPopup from 'components/ToastPopup'
import PrintControlBar from './components/PrintControlBar'
import ProductQuantityPopup from './components/ProductQuantityPopup'
import ProductList from './components/ProductList'
import { useTouenRefreshWithMinDisplay } from '@/hooks/useTouenRefreshWithMinDisplay'

export default function TouenCount() {
  // 1) 初期データ
  const {
    sessionUserName,
    customers,
    nearestClientName,
    setNearestClientName,
    sortedItems,
    moyoriSaki,
    onLocateNearest,
    products,
    setProducts,
    setProductsMap,
    itemObject,
    setItemObject,
    touenItems,
    setTouenItems,
    previewRowsOnce,
  } = useTouenInitializer()

  // 2) 商品取得
  const { listLoading, stableFetchComplete, forceRefresh } = useFetchTouenItems({
    nearestClientName,
    previewRowsOnce,
    customers,
    touenItems,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
    // oneShotPerClient は true のまま（自動fetchの暴発を抑える）
  })

  // 3) 印刷
  const {
    printStatus,
    setPrintStatus,
    showToast,
    setShowToast,
    second,
    setSecond,
    handleClick,
    onCancelPrint,
    isPrintingRetrying,
    setPrintDisabled,
    isDisabled,
    buttonStyle,
  } = usePrintTouenOrder({
    nearestClientName: nearestClientName ?? '',
    products,
    itemObject,
    sessionUserName,
  })

  // 4) +/- 操作
  const isIncrement = createIsIncrement({
    hapticOn,
    setProducts,
    nearestClientName: nearestClientName ?? '',
    setProductsMap,
  })

  // 5) 長押しポップアップ
  const {
    selectedProduct,
    setSelectedProduct,
    showPopup,
    setShowPopup,
    inputRef,
    handlePressStart,
    handlePressEnd,
    itemQtySave,
  } = useManageTouenPopup({
    setProducts,
    setSecond,
    setPrintStatus,
    setShowToast,
  })

  // 6) UI補助
  const todayTime = useNowTime(5000)
  const [modalVisible, setModalVisible] = useState(false)
  const [searchText, setSearchText] = useState('')

  useLockBodyScroll()

  const [mounted, setMounted] = useState(false)
  useEffect(() => {
    setMounted(true)
  }, [])

  // 7) 選択時に必ず最新取得（옵션 B 핵심）
  const handleSelectClient = useCallback(
    async (name: string) => {
      // 同一名でも「再選択」され得るため、ここで必ず fetch を叩く
      setNearestClientName(name)
      await forceRefresh(name)
    },
    [setNearestClientName, forceRefresh]
  )

  const popupElement =
    showPopup && selectedProduct ? (
      <ProductQuantityPopup
        showPopup={showPopup}
        setShowPopup={setShowPopup}
        selectedProduct={selectedProduct}
        setSelectedProduct={setSelectedProduct}
        inputRef={inputRef}
        itemQtySave={itemQtySave}
      />
    ) : null

  const clientSearchModalElement = modalVisible ? (
    <ClientSearchModal
      searchText={searchText}
      setSearchText={setSearchText}
      sortedItems={sortedItems}
      // モーダル側でも選択=最新取得に寄せるため、setNearestClientName だけ渡すのはやめる
      setNearestClientName={async (name: string) => {
        await handleSelectClient(name)
      }}
      setModalVisible={setModalVisible}
      userLatitude={null}
      userLongitude={null}
      currentSelectedName={nearestClientName ?? undefined}
    />
  ) : null

  const toastElement = showToast ? (
    <ToastPopup message={printStatus} setToast={setShowToast} position="center" second={second} />
  ) : null

  const printCancelButtonElement = isPrintingRetrying ? (
    <div className="fixed bottom-24 left-1/2 -translate-x-1/2 z-[9999] text-white">
      <button
        type="button"
        onClick={onCancelPrint}
        className="px-5 py-2 rounded-full bg-black/90 border border-gray-300 text-white shadow-md backdrop-blur-md active:translate-y-px"
      >
        キャンセル
      </button>
    </div>
  ) : null

  // 下スワイプ更新：選択中の得意先で強制更新
  const { handleRefresh } = useTouenRefreshWithMinDisplay(async () => {
    if (!nearestClientName) return
    await forceRefresh(nearestClientName)
  }, 1000)

  return (
    <div className="bg-gray-100 overflow-hidden select-none flex flex-1 flex-col">
      {popupElement}
      {clientSearchModalElement}

      <div className="p-2 md:h-auto">
        {toastElement}
        {printCancelButtonElement}

        <ClientSelectorBox
          nearestClientName={nearestClientName ?? undefined}
          nearestClientData={moyoriSaki ?? ''}
          onOpenModal={() => setModalVisible(true)}
          // 選択イベントで必ず最新取得
          onClientNameChange={async name => {
            await handleSelectClient(name)
          }}
          // 現在位置から最寄りを再計算 → 再計算後に最新取得
          onLocateNearest={async () => {
            await onLocateNearest()
            if (nearestClientName) {
              await forceRefresh(nearestClientName)
            }
          }}
        />

        <div className="flex justify-center content-center mt-2 text-sm font-semibold bg-gray-200 border border-gray-300 w-[98%] text-center p-1 rounded-lg">
          {todayTime}
        </div>

        <ProductList
          products={products}
          listLoading={listLoading}
          stableFetchComplete={stableFetchComplete}
          handlePressStart={handlePressStart}
          handlePressEnd={handlePressEnd}
          isIncrement={isIncrement}
          handleKeyDown={() => {}}
          onSwipeUpRefresh={handleRefresh}
        />

        <PrintControlBar
          handleClick={handleClick}
          isDisabled={isDisabled}
          buttonStyle={buttonStyle}
          onChangePrintDisabled={setPrintDisabled}
        />
      </div>
    </div>
  )
}

```

```
'use client'

import { useCallback, useEffect, useRef, useState } from 'react'
import type { Dispatch, SetStateAction } from 'react'
import { useErrorBoundary } from 'react-error-boundary'
import { TOUEN_NEW_ERROR_MESSAGES } from '@/constants/api/touen'

import useTouenCount from './useTouenCountActions'
import type { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

import { getInitialProductsForClient } from '../lib/getInitialProductsForClient'

// 元ロジック依存
import { alphabetClass, numberClass } from '../services/classesConstants'
import { fetchDetail } from '../services/steptaskDetailService'
import { normalizePrice } from '../services/format'

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
 * StepTask item 加工（元の getSteptaskSyouhin 内の加工を移管）
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

    const alphaFirstKeys = alphabetClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })
    const alphaSecondKeys = numberClass.filter(k => {
      const v = classHash?.[k]
      return v !== undefined && v !== null && String(v).trim() !== ''
    })

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

    const draftPakuCustomHash: Record<string, string | null> = {}
    const draftPakuCustomHashTwo: Record<string, string> = {}
    const draftPakuCustomHashThree: Record<string, string> = {}
    const draftPakuCustomHashFour: Record<string, string> = {}
    const draftClassHash: Record<string, string> = {}
    const draftPakuCustomHashProductIndex: Record<string, number> = {}
    const draftPakuCustomHashMasterIndex: Record<string, number> = {}

    const tasksForItem: Promise<void>[] = []

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

    Object.assign(item.PakuCustomHash as Record<string, string | null>, draftPakuCustomHash)
    Object.assign(item.PakuCustomHashTwo as Record<string, string>, draftPakuCustomHashTwo)
    Object.assign(item.PakuCustomHashThree as Record<string, string>, draftPakuCustomHashThree)
    Object.assign(item.PakuCustomHashFour as Record<string, string>, draftPakuCustomHashFour)

    Object.assign(item.ClassHash as Record<string, string>, draftClassHash)

    Object.assign(item.PakuCustomHashProductIndex as Record<string, number>, draftPakuCustomHashProductIndex)
    Object.assign(item.PakuCustomHashMasterIndex as Record<string, number>, draftPakuCustomHashMasterIndex)
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

  /** 得意先別の実行済み指標（previewTs または Date.now を保存） */
  const ranForClientRef = useRef<Map<string, number>>(new Map())

  /** 同一得意先の多重 fetch を防止 */
  const inFlightRef = useRef<Set<string>>(new Set())

  /** 既存 touenItems を最新として参照するための ref */
  const touenItemsRef = useRef<SteptaskItem[]>([])
  useEffect(() => {
    touenItemsRef.current = Array.isArray(touenItems) ? touenItems : []
  }, [touenItems])

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

  /** 最寄り先が変わるたびに実行済みを解除（自動 fetch を許可） */
  useEffect(() => {
    if (nearestClientName) {
      ranForClientRef.current.delete(nearestClientName)
    }
  }, [nearestClientName])

  /**
   * ページ全取得(Title ExactMatch)
   * - API は「ページ raw」を返す想定
   */
  const fetchAllByClient = useCallback(
    async (clientName: string): Promise<SteptaskItem[]> => {
      const pageSize = 200

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

      await enrichItemsInPlace(list)

      return list
    },
    [getSteptaskSyouhin]
  )

  /**
   * 明示的な強制更新（選択イベントから呼ぶ）
   * - 常に最新取得
   * - inFlight と markRan を先に入れて重複を抑止
   */
  const forceRefresh = useCallback(
    async (clientName?: string) => {
      if (!clientName || clientName === '最寄り先を選択') return
      if (inFlightRef.current.has(clientName)) return

      inFlightRef.current.add(clientName)
      ranForClientRef.current.set(clientName, Date.now())

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
   * 自動 fetch（初回・最寄り変更時の保険）
   * - 選択イベント側(forceRefresh)が主で、ここは補助
   */
  useEffect(() => {
    if (!nearestClientName || nearestClientName === '最寄り先を選択') return
    if (!Array.isArray(customers) || customers.length === 0) return
    if (inFlightRef.current.has(nearestClientName)) return

    const rows: PreviewRow[] = previewRowsOnce ?? []
    const rowForClient = rows.find(r => {
      const title = r?.タイトル ?? r?.Title ?? ''
      return String(title).trim() === String(nearestClientName).trim()
    })

    const currentPreviewTs = ts(rowForClient?.更新日時 ?? rowForClient?.UpdatedTime ?? rowForClient?.updatedTime)
    const lastTs = ranForClientRef.current.get(nearestClientName) ?? -1

    // one-shot: 最新ならスキップ（previewTs が 0 でもスキップ可能）
    if (oneShotPerClient && lastTs >= currentPreviewTs) {
      // 画面表示用 products が空の場合は手元データから補完のみ行う
      ;(async () => {
        try {
          const source = touenItemsRef.current
          if (!Array.isArray(source) || source.length === 0) return

          const products = await getInitialProductsForClient(
            nearestClientName,
            customers,
            source,
            obj => setItemObject(obj),
            source
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

    // 自動側も多重防止
    inFlightRef.current.add(nearestClientName)
    ranForClientRef.current.set(nearestClientName, currentPreviewTs || Date.now())

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
    previewRowsOnce,
    customers,
    oneShotPerClient,
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
