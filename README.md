```
useEffect(() => {
  const name = (nearestClientName ?? '').trim()
  if (!name || name === '最寄り先を選択') return
  if (!Array.isArray(customers) || customers.length === 0) return

  const rows: PreviewRow[] = previewRowsOnce ?? []
  const rowForClient = rows.find(r => {
    const title = (r?.タイトル ?? r?.Title ?? '').trim()
    return title === name
  })

  const previewTs = ts(
    rowForClient?.更新日時 ??
      rowForClient?.UpdatedTime ??
      rowForClient?.updatedTime
  )

  // ✅ IIFE 방식 (명시적으로 async 흐름을 보여줌)
  ;(async () => {
    await fetchClient(name, {
      previewTs,
      force: false,
    })
  })()

}, [
  nearestClientName,
  previewRowsOnce,
  customers.length, // 배열 전체가 아니라 length만
  fetchClient,
])

```
