```
'use client'

import { useEffect, useRef, useState, useCallback, useMemo, MouseEvent } from 'react'
import { cancelPrint } from 'lib/commonTouen/cancelPrint'
import { v4 as uuidv4 } from 'uuid'
import { Product } from '@/api/loadCustomerData'
import { useRouter } from 'next/router'
import useLoading from '@/hooks/useLoading'
import useTouenCount from './useTouenCountActions'
import {
  buildPrintDescription,
  compareIdAsc,
  formatCurrentTime,
  hasWebkitPrintHandler,
  mergeItems,
} from '../lib/utils'
import { headerPaths } from '@/lib/paths'
import useToast from '@/hooks/useToast'
import { PrintResultWindow } from '../types/types'
import { PRINT_MESSAGES } from '@/constants/messages/touen'
import { COMMON_MESSAGES } from '@/constants/messages/common'
import { hapticOn } from '@/components/hapticOn'
import { isIOS } from '../../list/services/utils'

/** ---- 定数（マジックナンバーの集約） ---- */
const RETRY_INTERVAL_MS = 5000
const SEND_COOLDOWN_MS = 2000
const ACK_GRACE_MS = 1200
const MAX_RETRIES = 19 // 初回送信 + 18回
const GLOBAL_TIMEOUT_MS = 90_000
import { ObjectResult } from '../types/types'

export const usePrintTouenOrder = ({
  nearestClientName,
  products,
  itemObject,
  sessionUserName,
}: {
  nearestClientName: string
  products: Product[]
  itemObject: ObjectResult
  sessionUserName: string | null
}) => {
  const router = useRouter()
  const { setLoading } = useLoading()
  const { openToast } = useToast()
  const { createTouenCount, getTouenClientId } = useTouenCount()
  // 状態
  const [isProcessing, setIsProcessing] = useState(false)
  const [second, setSecond] = useState<number>(4800)
  const [printStatus, setPrintStatus] = useState<string>('')
  const [showToast, setShowToast] = useState(false)
  const [isPrintingRetrying, setIsPrintingRetrying] = useState(false)
  const [printDisabled, setPrintDisabled] = useState(false)

  // 再試行タイマー／キャンセルフラグ

  const retryIntervalRef = useRef<ReturnType<typeof setInterval> | null>(null)
  const printCancelledRef = useRef(false)

  // クリーンアップ
  useEffect(() => {
    return () => {
      if (retryIntervalRef.current) {
        clearInterval(retryIntervalRef.current)
        retryIntervalRef.current = null
      }
    }
  }, [])

  // キャンセル
  const onCancelPrint = useCallback(() => {
    cancelPrint(
      printCancelledRef,
      retryIntervalRef,
      setIsPrintingRetrying,
      setLoading,
      setSecond,
      setPrintStatus,
      setShowToast
    )
  }, [])

  const handleClick = useCallback(
    async (e: MouseEvent<HTMLButtonElement>) => {
      const ts = Date.now().toString().slice(-2) // キャッシュ使わないためのクエリパラメータ

      const filtered = products.filter(p => p.quantity > 0)
      if (filtered.length === 0) return
      const touenClientCode = await getTouenClientId(itemObject.ResultID)
      if (!touenClientCode) return //do something

      const sortedById = [...filtered].sort(compareIdAsc)
      const nowTime = formatCurrentTime()
      const merged = mergeItems(
        sortedById,
        nearestClientName,
        itemObject as Record<string, string>,
        nowTime
      )
      // const printStatusMessages = resolveMessages(messages)
      const batchId = uuidv4()

      const payload = {
        Data: merged.map(item => ({
          Title: item.productName || 'タイトル無し',
          DescriptionHash: {
            DescriptionA: String(touenClientCode),
            DescriptionB: String(item.clientName).trim(),
            DescriptionC: String(item.storageCode),
            DescriptionD: String(item.productNumber),
            DescriptionE: String(item.productName),
            DescriptionG: String(item.destination_code),
          },
          DateHash: {
            DateA: item.nowTime,
          },
          NumHash: {
            NumA: Number(item.quantity),
            NumB: Number(item.productPrice), //re-check price
            NumC: Number(item.id), // 並び順
          },
          ClassHash: {
            ClassY: batchId,
            ClassZ: sessionUserName ?? '',
          },
        })),
      }
      //print disable == true -> just call api then navigate
      if (printDisabled) {
        setLoading(true)
        const response = await createTouenCount(payload)
        if (response?.code === 200) {
          openToast.success(COMMON_MESSAGES.SUCCESS, 'center')
          if (typeof window !== 'undefined') {
            window.dispatchEvent(new Event('pipit:refreshTouenHistory'))
          }
          setLoading(false) // untils use loading context should be put under navigated page
          router.push(`${headerPaths.touenList}?nc=${ts}`)
        } else {
          setLoading(false)
        }
      } else {
        const win = typeof window !== 'undefined' ? window : undefined
        // recheck ios env
        if (!isIOS || !hasWebkitPrintHandler(win as PrintResultWindow)) {
          setLoading(false)
          // reject
          openToast.error(PRINT_MESSAGES.NOT_IOS_FAIL, 'center')
        } else {
          //print
          const printDescription = buildPrintDescription(nowTime, nearestClientName, sortedById)

          if (retryIntervalRef?.current) {
            clearInterval(retryIntervalRef.current)
            retryIntervalRef.current = null
          }
          if (printCancelledRef) printCancelledRef.current = false
          let retryCount = 0
          const startTime = Date.now()
          let handled = false
          let waitingAck = false
          let lastSentAt = 0
          let ackLockUntil = 0

          const stopRetrying = () => {
            if (retryIntervalRef?.current) {
              clearInterval(retryIntervalRef.current)
              retryIntervalRef.current = null
            }
            setIsPrintingRetrying(false)
          }
          const tryPrint = () => {
            if (handled) return
            if (printCancelledRef?.current) {
              stopRetrying()
              return
            }
            if (waitingAck) return
            const now = Date.now()
            if (now < ackLockUntil) return
            if (now - lastSentAt < SEND_COOLDOWN_MS) return

            // グローバルタイムアウト
            if (now - startTime > GLOBAL_TIMEOUT_MS) {
              setSecond(3000)
              setPrintStatus(PRINT_MESSAGES.NOT_AVAILABLE ?? 'プリンタが見つかりませんでした。')
              setShowToast(true)
              setLoading(false)
              hapticOn('error')
              stopRetrying()
              waitingAck = false
              return
            }

            if (retryCount >= MAX_RETRIES) {
              setSecond(0)
              setLoading(false)
              setPrintStatus('')
              setShowToast(false)
              stopRetrying()
              return
            }

            if (hasWebkitPrintHandler(win)) {
              retryCount++
              const remaining = MAX_RETRIES - retryCount
              waitingAck = true
              lastSentAt = now
              win!.webkit!.messageHandlers!.printHandler!.postMessage(printDescription)
              hapticOn('medium')
              // uiSetWaiting(ui, printStatusMessages.PRINT_WAIT, remaining, 4900)
              openToast.info(`${PRINT_MESSAGES.WAIT} ${remaining}`, 'center')
              setSecond(4900)
            }
          }
          setIsPrintingRetrying(true)
          setLoading(true)
          tryPrint()
          if (retryIntervalRef) {
            retryIntervalRef.current = setInterval(tryPrint, RETRY_INTERVAL_MS)
          }

          if (win) {
            win.onPrintResult = async (message: string) => {
              if (handled) return
              if (printCancelledRef?.current) {
                stopRetrying()
                waitingAck = false
                return
              }

              const isSuccess = typeof message === 'string' && message.includes('✅')
              if (isSuccess) {
                handled = true
                stopRetrying()
                ;(win as PrintResultWindow).onPrintResult = null
                waitingAck = false

                const response = await createTouenCount(payload)
                if (response?.code === 200) {
                  openToast.success(PRINT_MESSAGES.SUCCESS, 'center')
                  if (typeof window !== 'undefined') {
                    window.dispatchEvent(new Event('pipit:refreshTouenHistory'))
                  }
                  router.push(`${headerPaths.touenList}?nc=${ts}`)
                  setLoading(false) // untils use loading context should be put under navigated page
                }
                // 印刷・実績登録同時に実行する
                // await Promise.all([
                //   postHistoryAndPrint(merged, nowTime, sessionUserName, Identifier).catch(err =>
                //     console.error('登録失敗:', err)
                //   ),
                //   (uiSetSuccess(ui, printStatusMessages.PRINT_SUCCESS),
                //   ui?.refreshHistory?.(),
                //   ui?.push?.(nextUrl)),
                // ])
              } else {
                // 軽い待機ロック（連打レース対策）
                ackLockUntil = Date.now() + ACK_GRACE_MS
                setTimeout(() => {
                  waitingAck = false
                }, ACK_GRACE_MS)
                const remaining = MAX_RETRIES - retryCount
                if (remaining <= 0) {
                  setSecond(0)
                  setLoading(false)
                  setPrintStatus('')
                  setShowToast(false)
                  stopRetrying()
                  return
                }
                // uiSetWaiting(ui, printStatusMessages.PRINT_WAIT, remaining, 4800)
                openToast.info(`${PRINT_MESSAGES.WAIT} ${remaining}`, 'center')
                setLoading(true)
              }
            }
          }

          //api
        }
      }
    },
    [
      createTouenCount,
      getTouenClientId,
      itemObject,
      nearestClientName,
      openToast,
      printDisabled,
      products,
      router,
      sessionUserName,
      setLoading,
    ]
  )

  // ボタン状態／スタイル
  const isDisabled = useMemo(
    () =>
      isProcessing ||
      nearestClientName === '最寄り先を選択' ||
      !products.some(item => item.quantity > 0),
    [isProcessing, nearestClientName, products]
  )

  const buttonStyle = useMemo(
    () =>
      isDisabled
        ? 'bg-gray-400 cursor-not-allowed'
        : 'bg-blue-500 hover:bg-blue-600 hover:-translate-y-2 hover:scale-105 hover:shadow-2xl',
    [isDisabled]
  )

  return {
    // loading,
    setLoading,
    printStatus,
    setPrintStatus,
    showToast,
    setShowToast,
    second,
    setSecond,
    handleClick,
    onCancelPrint,
    isPrintingRetrying,
    printDisabled,
    setPrintDisabled,
    isDisabled,
    buttonStyle,
  }
}

````
