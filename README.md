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

    formattedItems.push(`${codeValue}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`)
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

    formattedItems.push(`${codeValue}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`)
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
