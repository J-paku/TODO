```
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
```

```
export const getSteptaskSyouhin = async (tokuisaki: string) => {
  const url = TOUEN_API_ENDPOINTS.ITEMS_GET
  const pageSize = 200

  try {
    // 初回：総件数確認
    const initialParams: SteptaskParams = {
      ApiVersion: 1.1,
      Offset: 0,
      PageSize: 1,
      View: {
        ColumnFilterHash: { Title: tokuisaki },
        ColumnFilterSearchTypes: { Title: 'ExactMatch' },
      },
    }
    if (isDebug) initialParams.ApiKey = ApiKey

    const initialRes = await postWithRetry(url, initialParams)
    const totalCount = initialRes.data.Response?.TotalCount
    if (typeof totalCount !== 'number') {
      throw new Error(`${ERR_PREFIX}TotalCountが不正です`)
    }

    // 全ページ取得（部分成功なし）
    const pageRequests: Promise<Awaited<ReturnType<typeof postWithRetry>>>[] = []
    for (let offset = 0; offset < totalCount; offset += pageSize) {
      const params = { ...initialParams, Offset: offset, PageSize: pageSize }
      pageRequests.push(postWithRetry(url, params))
    }
    const pageResponses = await Promise.all(pageRequests)
    const list = pageResponses.flatMap(res => res.data?.Response?.Data ?? [])

    // アイテム単位の加工（ドラフト→単一コミット）
    for (const item of list) {
      item.PakuCustomHash ||= {}
      item.PakuCustomHashTwo ||= {}
      item.PakuCustomHashThree ||= {}
      item.PakuCustomHashFour ||= {}
      item.PakuCustomSoko ||= {}
      item.ClassHash ||= {}
      item.PakuCustomHashProductIndex ||= {}
      item.PakuCustomHashMasterIndex ||= {}

      // 値ありキー抽出
      const alphaFirstKeys = alphabetClass.filter(k => {
        const v = item.ClassHash?.[k]
        return v !== undefined && v !== null && String(v).trim() !== ''
      })
      const alphaSecondKeys = numberClass.filter(k => {
        const v = item.ClassHash?.[k]
        return v !== undefined && v !== null && String(v).trim() !== ''
      })

      // 末尾値ありインデックス
      const alphabetMasterIndex =
        alphabetClass
          .map((k, idx) => ({ idx, v: item.ClassHash?.[k] }))
          .filter(({ v }) => String(v ?? '').trim() !== '')
          .at(-1)?.idx ?? -1

      const numberMasterIndex =
        numberClass
          .map((k, idx) => ({ idx, v: item.ClassHash?.[k] }))
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
        const value = item.ClassHash![key]
        const customKey = `Custom${String.fromCharCode(65 + idx)}`
        tasksForItem.push(
          (async () => {
            const d = await fetchDetail(value, idx)
            draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
            draftPakuCustomHashTwo[customKey] = d.destinationCode
            draftPakuCustomHashThree[customKey] = d.productName
            draftPakuCustomHashFour[customKey] = d.productCode
            draftClassHash[customKey] = d.sokoCode
            draftPakuCustomHashProductIndex[customKey] = d.productIndex
          })()
        )
      })

      // 数字側（Custom001〜）※ 連番継続: alphaFirstCount + idx
      alphaSecondKeys.forEach((key, idx) => {
        const value = item.ClassHash![key]
        const customKey = `Custom${String(idx + 1).padStart(3, '0')}`
        const productIdx = alphaFirstCount + idx
        tasksForItem.push(
          (async () => {
            const d = await fetchDetail(value, productIdx)
            draftPakuCustomHash[customKey] = normalizePrice(d.koyamaPrice)
            draftPakuCustomHashTwo[customKey] = d.destinationCode
            draftPakuCustomHashThree[customKey] = d.productName
            draftPakuCustomHashFour[customKey] = d.productCode
            draftClassHash[customKey] = d.sokoCode
            draftPakuCustomHashProductIndex[customKey] = d.productIndex
          })()
        )
      })

      // MasterIndex付番（A,B,C...は1〜10、001,002...は11〜）
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
        const value = item.ClassHash![key]
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
        const value = item.ClassHash![key]
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

      Object.assign(item.PakuCustomHash!, draftPakuCustomHash)
      Object.assign(item.PakuCustomHashTwo!, draftPakuCustomHashTwo)
      Object.assign(item.PakuCustomHashThree!, draftPakuCustomHashThree)
      Object.assign(item.PakuCustomHashFour!, draftPakuCustomHashFour)
      Object.assign(item.ClassHash!, draftClassHash)
      Object.assign(item.PakuCustomHashProductIndex!, draftPakuCustomHashProductIndex)
      Object.assign(item.PakuCustomHashMasterIndex!, draftPakuCustomHashMasterIndex)
    }

    return list
  } catch (err) {
    const e = err instanceof Error ? err : new Error(`${ERR_PREFIX}不明なエラー`)
    console.error(e)
    throw e
  }
}
```
