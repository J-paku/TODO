```
import type { Customer, Product } from '@/api/loadCustomerData'
import { ObjectResult, SteptaskItem } from '../types/types'

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
  const selectedClientItem = source?.find(
    (item: SteptaskItem) => item.ClassHash?.ClassA == tokuisakiID
  )
  if (!selectedClientItem) return []

  // PakuCustomHashTwo を追加（元のロジックを変えない）
  const {
    ClassHash,
    PakuCustomHash,
    PakuCustomHashTwo,
    PakuCustomHashThree,
    PakuCustomHashFour,
    PakuCustomHashMasterIndex,
  } = selectedClientItem as {
    ClassHash: { ClassA: string; ClassB: string; Custom001: string; CustomA: string }
    PakuCustomHash?: Record<string, string>
    PakuCustomHashTwo?: Record<string, string>
    PakuCustomHashThree: Record<string, string>
    PakuCustomHashFour: Record<string, string>
    PakuCustomHashMasterIndex: Record<string, number>
  }

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
    const customKey = `Custom${customAlphaList[i]}`

    const name = String(PakuCustomHashThree[customKey] ?? '').trim()
    const code = String(PakuCustomHashFour[customKey] ?? '').trim()

    const rawCustom = String(PakuCustomHash?.[customKey] ?? '').trim()
    const rawCustomTwo = String(PakuCustomHashTwo?.[customKey] ?? '').trim()

    const masterIndex = String(PakuCustomHashMasterIndex?.[customKey] ?? '').trim()

    let customValue = rawCustom
    let customValueTwo = rawCustomTwo
    let customMasterIndex = masterIndex

    if (!name && !code) {
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

    // custom2 が空なら直前の確定値で補完（スパースインデックス対策）
    if (!customValueTwo && lastCustomTwo) customValueTwo = lastCustomTwo

    // 確定後に lastCustomTwo を更新
    if (customValueTwo) lastCustomTwo = customValueTwo

    formattedItems.push(`${code}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`)
  }

  // ===== U〜Z のペア（Custom001〜） =====
  let pendingAlpha = ''
  let pendingAlphaTwo = ''
  let pendingAlphaMasterIndex = ''
  let customIndex = 1

  for (let ch = 'U'.charCodeAt(0); ch <= 'Z'.charCodeAt(0); ch += 2) {
    const customKey = `Custom${String(customIndex).padStart(3, '0')}`
    customIndex++

    const name = String(PakuCustomHashThree[customKey] ?? '').trim()
    const code = String(PakuCustomHashFour[customKey] ?? '').trim()

    const rawCustom = String(PakuCustomHash?.[customKey] ?? '').trim()
    const rawCustomTwo = String(PakuCustomHashTwo?.[customKey] ?? '').trim()

    const masterIndex = String(PakuCustomHashMasterIndex?.[customKey] ?? '').trim()

    let customValue = rawCustom
    let customValueTwo = rawCustomTwo
    let customMasterIndex = masterIndex

    if (!name && !code) {
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

    // custom2 が空なら直前の確定値で補完
    if (!customValueTwo && lastCustomTwo) customValueTwo = lastCustomTwo

    // 確定後に lastCustomTwo を更新
    if (customValueTwo) lastCustomTwo = customValueTwo

    formattedItems.push(`${code}, ${name}, ${customValue}, ${customValueTwo}, ${customMasterIndex}`)
  }

  // ===== デバッグ（Description001〜020 の 10件、Custom003〜） =====

  // 保留キュー（custom1/custom2/masterIndex）
  let pendC1Queue: string[] = []
  let pendC2Queue: string[] = []
  let pendMIQueue: string[] = []

  // 直前確定値（custom2/masterIndex）
  let lastC2 = ''
  let lastMI = ''

  // 値を文字列化し trim するユーティリティ
  const toStrTrim = (v: unknown): string =>
    typeof v === 'string' ? v.trim() : String(v ?? '').trim()

  for (let idx = 2; idx <= 10; idx++) {
    // Custom004 ～ Custom012 を想定（idx+2）
    const customKey = `Custom${String(idx + 2).padStart(3, '0')}`

    // 生値の正規化
    const rawName = toStrTrim(PakuCustomHashThree?.[customKey])
    const rawCode = toStrTrim(PakuCustomHashFour?.[customKey])
    const rawC1 = toStrTrim(PakuCustomHash?.[customKey])
    const rawC2 = toStrTrim(PakuCustomHashTwo?.[customKey])
    const rawMI = toStrTrim(PakuCustomHashMasterIndex?.[customKey])

    // name と code が両方空 → 値は保留して次へ（custom1/custom2/masterIndex を同じロジックで保留）
    if (!rawName && !rawCode) {
      if (rawC1) pendC1Queue.push(rawC1)
      if (rawC2) pendC2Queue.push(rawC2)
      if (rawMI) pendMIQueue.push(rawMI)
      continue
    }

    // キュー優先で確定値決定（FIFO）
    const finalC1 = pendC1Queue.length > 0 ? (pendC1Queue.shift() as string) : rawC1
    let finalC2 = pendC2Queue.length > 0 ? (pendC2Queue.shift() as string) : rawC2
    let finalMI = pendMIQueue.length > 0 ? (pendMIQueue.shift() as string) : rawMI

    // custom2 が空なら直前の確定値で補完
    if (!finalC2 && lastC2) {
      finalC2 = lastC2
    }
    // masterIndex が空なら直前の確定値で補完（他キーと同じロジック）
    if (!finalMI && lastMI) {
      finalMI = lastMI
    }

    // 出力は常に 5 カラム（code, name, custom1, custom2, masterIndex）
    formattedItems.push(`${rawCode}, ${rawName}, ${finalC1}, ${finalC2}, ${finalMI}`)

    // 元ロジック通り：未消化の生値をキューに戻す（確定値に使われなかった分を保留継続）
    if (rawC1 && rawC1 !== finalC1) pendC1Queue.push(rawC1)
    if (rawC2 && rawC2 !== finalC2) pendC2Queue.push(rawC2)
    if (rawMI && rawMI !== finalMI) pendMIQueue.push(rawMI)

    // 直前確定値の更新
    if (finalC2) lastC2 = finalC2
    if (finalMI) lastMI = finalMI
  }

  // ===== objectResult の構築（既存通り） =====
  const objectResult: ObjectResult = {
    ResultID: ClassHash.ClassA,
    storageCode: ClassHash.Custom001 ?? ClassHash.CustomA,
  }

  let validIndex = 1
  formattedItems.forEach(item => {
    const [productNumber, productName] = item?.split(',').map(str => str.trim()) ?? []
    const isValid = productNumber && productName && productNumber !== '' && productName !== ''

    if (isValid) {
      objectResult[`item${validIndex}`] = item // "code, name, custom1, custom2"
      validIndex++
    }
  })

  setItemObject(objectResult)

  // 画面返却：名称は 2列目のみ使用（既存通り）
  return formattedItems.map((item, index) => {
    const parts = item.split(',').map(s => s.trim())
    const [, name, , destinationCode, narabijyun] = parts

    // console.log('get Item Array', parts)
    return {
      id: index + 1,
      product_name: name ?? '',
      quantity: 0,
      destination_code: destinationCode ?? '',
      narabijyun: narabijyun ?? '',
    }
  })
}
// 得意先によって品物が異なるためのメソッド END

```
