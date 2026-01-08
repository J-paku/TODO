```
// React
import { useEffect, useState } from 'react'
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

/**
 * - ページ内の状態・印刷処理・モーダル・数入力ポップアップなどを統合管理する
 * - SSR(初期HTML生成)とクライアント側の相互作用を切り分けるため、表示系の一部はマウント後に制御する
 */
export default function TouenCount() {
  // --- 1. 初期データ / ユーザー・顧客・位置情報のセットアップ ---
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

  // --- 2. 登園商品の取得と読み込み状態 ---
  const { listLoading, stableFetchComplete, forceRefresh } = useFetchTouenItems({
    nearestClientName,
    previewRowsOnce,
    customers,
    touenItems,
    setTouenItems,
    setItemObject,
    setProducts,
    setProductsMap,
  })

  // --- 3. 印刷処理の管理 ---
  const {
    // loading,
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

  // --- 4. 商品数の増減(+/-)処理 ---
  const isIncrement = createIsIncrement({
    hapticOn,
    setProducts,
    nearestClientName: nearestClientName ?? '',
    setProductsMap,
  })

  // --- 5. 長押し・数量入力ポップアップの管理 ---
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

  // --- 6. トースト・モーダル・時間表示 ---
  const todayTime = useNowTime(5000)
  const [modalVisible, setModalVisible] = useState(false)
  const [searchText, setSearchText] = useState('')

  // iOS WKWebView 対策：ページ滞在中は body/html のスクロールを固定
  useLockBodyScroll()

  // クライアントマウント判定：ブラウザ限定UIの表示に利用
  const [mounted, setMounted] = useState(false)
  useEffect(() => {
    setMounted(true)
  }, [])

  // --- 7. 条件付きUI要素の作成 ---
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
      setNearestClientName={setNearestClientName}
      setModalVisible={setModalVisible}
      userLatitude={null}
      userLongitude={null}
      currentSelectedName={nearestClientName ?? undefined}
    />
  ) : null

  const toastElement = showToast ? (
    <ToastPopup message={printStatus} setToast={setShowToast} position='center' second={second} />
  ) : null

  const printCancelButtonElement = isPrintingRetrying ? (
    <div className='fixed bottom-24 left-1/2 -translate-x-1/2 z-[9999] text-white'>
      <button
        type='button' // フォーム送信を防ぐための明示的なタイプ指定
        onClick={onCancelPrint}
        className='px-5 py-2 rounded-full bg-black/90 border
          border-gray-300 text-white shadow-md backdrop-blur-md active:translate-y-px'
      >
        🖨️ キャンセル
      </button>
    </div>
  ) : null

  const { handleRefresh } = useTouenRefreshWithMinDisplay(async () => {
    await forceRefresh(nearestClientName ?? undefined)
  }, 1000)

  // --- 8. JSXレンダー ---
  return (
    <div className='bg-gray-100 overflow-hidden select-none flex flex-1 flex-col'>
      {/* {loading && mounted ? <LoadingModal /> : null} */}
      {popupElement}
      {clientSearchModalElement}

      <div className='p-2 md:h-auto'>
        {toastElement}
        {printCancelButtonElement}

        <ClientSelectorBox
          nearestClientName={nearestClientName ?? undefined}
          nearestClientData={moyoriSaki ?? ''}
          onOpenModal={() => setModalVisible(true)}
          onClientNameChange={name => setNearestClientName(name)}
          onLocateNearest={onLocateNearest}
        />

        <div
          className='
            flex justify-center content-center mt-2 text-sm font-semibold 
            bg-gray-200 border border-gray-300 w-[98%] text-center p-1 rounded-lg
          '
        >
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
import type { Customer, Product } from '@/api/loadCustomerData'
import type { ObjectResult, SteptaskItem } from '../types/types'

// 得意先によって品物が異なるためのメソッド
export const getInitialProductsForClient = async (
  clientName: string,
  customers: Customer[],
  touenItems: SteptaskItem[],
  setItemObject: (obj: ObjectResult) => void,
  itemsOverride?: SteptaskItem[]
): Promise<Product[]> => {
  const tokuisakiIdFinder = customers.find(
    el => String(el.得意先名).trim() === String(clientName).trim()
  )
  const tokuisakiID = tokuisakiIdFinder?.ID

  const source = itemsOverride ?? touenItems

  const selectedClientItem = source?.find(item => {
    // ClassA 가 undefined일 수 있으므로 안전하게 문자열 비교
    const classA = item?.ClassHash?.ClassA
    return classA != null && tokuisakiID != null && String(classA) === String(tokuisakiID)
  })

  if (!selectedClientItem) return []

  // ====== 여기부터 "undefined object" 방어가 핵심 ======
  const ClassHash = selectedClientItem.ClassHash ?? ({} as Record<string, string>)

  const PakuCustomHash = selectedClientItem.PakuCustomHash ?? {}
  const PakuCustomHashTwo = selectedClientItem.PakuCustomHashTwo ?? {}
  const PakuCustomHashThree = selectedClientItem.PakuCustomHashThree ?? {}
  const PakuCustomHashFour = selectedClientItem.PakuCustomHashFour ?? {}
  const PakuCustomHashMasterIndex = selectedClientItem.PakuCustomHashMasterIndex ?? {}
  // ====================================================

  const formattedItems: string[] = []

  const startCharCode = 'A'.charCodeAt(0)
  const endCharCode = 'U'.charCodeAt(0)
  const alphaList: string[] = []
  const customAlphaList: string[] = []

  for (let code = startCharCode; code < endCharCode; code += 2) {
    alphaList.push(String.fromCharCode(code))
  }
  for (let code = startCharCode; code < endCharCode; code++) {
    customAlphaList.push(String.fromCharCode(code))
  }

  let pendingCustom = ''
  let pendingCustomTwo = ''
  let pendingMasterIndex = ''
  let lastCustomTwo = ''

  // ===== A〜U のペア（CustomA〜） =====
  for (let i = 0; i < alphaList.length; i++) {
    const code = alphaList[i]
    const customKey = `Custom${customAlphaList[i]}`

    const name = String(PakuCustomHashThree?.[customKey] ?? '').trim()
    const codeValue = String(PakuCustomHashFour?.[customKey] ?? '').trim()

    const rawCustom = String(PakuCustomHash?.[customKey] ?? '').trim()
    const rawCustomTwo = String(PakuCustomHashTwo?.[customKey] ?? '').trim()
    const masterIndex = String(PakuCustomHashMasterIndex?.[customKey] ?? '').trim()

    let customValue = rawCustom
    let customValueTwo = rawCustomTwo
    let customMasterIndex = masterIndex

    if (!name && !codeValue) {
      if (rawCustom) pendingCustom = rawCustom
      if (rawCustomTwo) pendingCustomTwo = rawCustomTwo
      if (masterIndex) pendingMasterIndex = masterIndex
      continue
    }

    if (pendingCustom) {
      customValue = pendingCustom
      pendingCustom = rawCustom || ''
    }
    if (pendingCustomTwo) {
      customValueTwo = pendingCustomTwo
      pendingCustomTwo = rawCustomTwo || ''
    }
    if (pendingMasterIndex) {
      customMasterIndex = pendingMasterIndex
      pendingMasterIndex = masterIndex || ''
    }

    if (!customValueTwo && lastCustomTwo) customValueTwo = lastCustomTwo
    if (customValueTwo) lastCustomTwo = customValueTwo

    formattedItems.push(
      `${codeValue}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`
    )
  }

  // ===== U〜Z のペア（Custom001〜） =====
  let pendingAlpha = ''
  let pendingAlphaTwo = ''
  let pendingAlphaMasterIndex = ''
  let customIndex = 1

  for (let ch = 'U'.charCodeAt(0); ch <= 'Z'.charCodeAt(0); ch += 2) {
    const code = String.fromCharCode(ch)
    const customKey = `Custom${String(customIndex).padStart(3, '0')}`
    customIndex++

    const name = String(PakuCustomHashThree?.[customKey] ?? '').trim()
    const codeValue = String(PakuCustomHashFour?.[customKey] ?? '').trim()

    const rawCustom = String(PakuCustomHash?.[customKey] ?? '').trim()
    const rawCustomTwo = String(PakuCustomHashTwo?.[customKey] ?? '').trim()
    const masterIndex = String(PakuCustomHashMasterIndex?.[customKey] ?? '').trim()

    let customValue = rawCustom
    let customValueTwo = rawCustomTwo
    let customMasterIndex = masterIndex

    if (!name && !codeValue) {
      if (rawCustom) pendingAlpha = rawCustom
      if (rawCustomTwo) pendingAlphaTwo = rawCustomTwo
      if (masterIndex) pendingAlphaMasterIndex = masterIndex
      continue
    }

    if (pendingAlpha) {
      customValue = pendingAlpha
      pendingAlpha = rawCustom || ''
    }
    if (pendingAlphaTwo) {
      customValueTwo = pendingAlphaTwo
      pendingAlphaTwo = rawCustomTwo || ''
    }
    if (pendingAlphaMasterIndex) {
      customMasterIndex = pendingAlphaMasterIndex
      pendingAlphaMasterIndex = masterIndex || ''
    }

    if (!customValueTwo && lastCustomTwo) customValueTwo = lastCustomTwo
    if (customValueTwo) lastCustomTwo = customValueTwo

    formattedItems.push(
      `${codeValue}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`
    )
  }

  // ===== デバッグ（Description001〜020 の 10件、Custom003〜） =====
  let pendC1Queue: string[] = []
  let pendC2Queue: string[] = []
  let pendMIQueue: string[] = []

  let lastC2 = ''
  let lastMI = ''

  const toStrTrim = (v: unknown): string =>
    typeof v === 'string' ? v.trim() : String(v ?? '').trim()

  for (let idx = 2; idx <= 10; idx++) {
    const customKey = `Custom${String(idx + 2).padStart(3, '0')}`

    const rawName = toStrTrim(PakuCustomHashThree?.[customKey])
    const rawCode = toStrTrim(PakuCustomHashFour?.[customKey])
    const rawC1 = toStrTrim(PakuCustomHash?.[customKey])
    const rawC2 = toStrTrim(PakuCustomHashTwo?.[customKey])
    const rawMI = toStrTrim(PakuCustomHashMasterIndex?.[customKey])

    if (!rawName && !rawCode) {
      if (rawC1) pendC1Queue.push(rawC1)
      if (rawC2) pendC2Queue.push(rawC2)
      if (rawMI) pendMIQueue.push(rawMI)
      continue
    }

    const finalC1 = pendC1Queue.length > 0 ? (pendC1Queue.shift() as string) : rawC1
    let finalC2 = pendC2Queue.length > 0 ? (pendC2Queue.shift() as string) : rawC2
    let finalMI = pendMIQueue.length > 0 ? (pendMIQueue.shift() as string) : rawMI

    if (!finalC2 && lastC2) finalC2 = lastC2
    if (!finalMI && lastMI) finalMI = lastMI

    formattedItems.push(`${rawCode}, ${rawName}, ${finalC1}, ${finalC2}, ${finalMI}`)

    if (rawC1 && rawC1 !== finalC1) pendC1Queue.push(rawC1)
    if (rawC2 && rawC2 !== finalC2) pendC2Queue.push(rawC2)
    if (rawMI && rawMI !== finalMI) pendMIQueue.push(rawMI)

    if (finalC2) lastC2 = finalC2
    if (finalMI) lastMI = finalMI
  }

  // ===== objectResult の構築（既存通り） =====
  const objectResult: ObjectResult = {
    ResultID: String(ClassHash?.ClassA ?? ''),
    storageCode: String((ClassHash as any)?.Custom001 ?? (ClassHash as any)?.CustomA ?? ''),
  }

  let validIndex = 1
  formattedItems.forEach(item => {
    const [productNumber, productName] = item?.split(',').map(str => str.trim()) ?? []
    const isValid = productNumber && productName && productNumber !== '' && productName !== ''
    if (isValid) {
      objectResult[`item${validIndex}`] = item
      validIndex++
    }
  })

  setItemObject(objectResult)

  return formattedItems.map((item, index) => {
    const parts = item.split(',').map(s => s.trim())
    const [, name, , destinationCode, narabijyun] = parts
    return {
      id: index + 1,
      product_name: name ?? '',
      quantity: 0,
      destination_code: destinationCode ?? '',
      narabijyun: narabijyun ?? '',
    }
  })
}

```
