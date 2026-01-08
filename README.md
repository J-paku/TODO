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
