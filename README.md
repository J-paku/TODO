```
'use client'

import React, { useEffect, useRef, useState, useMemo, useCallback, useLayoutEffect } from 'react'
import { COMMON_MESSAGES } from '@/constants/messages/common'
import { refreshOffset } from '@/lib/commonTouen/refreshOffset'
import { ClaimClient } from '@/api/loadCustomerData'
import { hapticOn } from './hapticOn'

/** 行データの共通型 */
type BaseRow = {
  creator?: string
  [key: string]: any
}

/** 汎用の列定義 */
export interface HistoryTableColumn<T extends BaseRow = BaseRow> {
  key: keyof T & string
  label: string
  align?: 'left' | 'center' | 'right'
  render?: (value: T[keyof T], row: T) => React.ReactNode
  headerClassName?: string
  cellClassName?: string
}

/** おむつ履歴用の行型 */
type TouenRow = {
  clients_name: string
  created_date: string
  created_time: string
  creator?: string
}

/** おむつ履歴テーブルの列定義（幅: 60% / 25% / 15%） */
export const touenListColumns: HistoryTableColumn<TouenRow>[] = [
  {
    key: 'clients_name',
    label: '得意先名',
    align: 'center',
    headerClassName: 'border border-gray-600 w-[60%]',
    cellClassName:
      'p-2 text-center border border-black truncate max-w-[150px] sm:max-w-none text-sm',
  },
  {
    key: 'created_date',
    label: '日付',
    align: 'center',
    headerClassName: 'border border-gray-600 w-[25%]',
    cellClassName: 'text-center border border-gray-600 text-xs',
  },
  {
    key: 'created_time',
    label: '時間',
    align: 'center',
    headerClassName: 'border border-gray-600 w-[15%]',
    cellClassName: 'text-center border border-gray-600 text-xs',
  },
]

/** クレーム履歴テーブルの列定義（元コードの min-w を維持しつつ固定幅化させる前提） */
export const claimHistoryColumns: HistoryTableColumn<ClaimClient>[] = [
  {
    key: 'status',
    label: '緊急度',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[15%]',
    cellClassName:
      'text-center border border-gray-300 truncate max-w-[100px] sm:max-w-none text-sm ',
    render: value => {
      const text = String(value ?? '')
      const isUrgent = text.includes('急')
      return <span className={isUrgent ? 'font-bold text-red-600' : undefined}>{text}</span>
    },
  },
  {
    key: 'clients_name',
    label: '得意先名',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[120px]',
    cellClassName: 'text-center border border-gray-300 truncate sm:max-w-none text-xs',
  },
  {
    key: 'urgency',
    label: '状況',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[5px]',
    cellClassName: 'text-center border border-gray-300 truncate sm:max-w-none font-bold text-sm ',
  },
  {
    key: 'category',
    label: 'カテゴリー',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[35px] whitespace-nowrap',
    cellClassName: 'text-center border border-gray-300 text-xs',
  },
  {
    key: 'claimContent',
    label: 'クレーム内容',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[50px] whitespace-nowrap',
    cellClassName: 'text-center border border-gray-300 text-xs',
  },
  {
    key: 'service',
    label: 'サービス',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[95px] whitespace-nowrap',
    cellClassName: 'text-center border border-gray-300 text-xs',
  },
  {
    key: 'serviceShohin',
    label: 'サービス内容',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[50px] whitespace-nowrap',
    cellClassName: 'text-center border border-gray-300 text-xs',
  },
  {
    key: 'created_date',
    label: '日付',
    align: 'center',
    headerClassName: 'border border-gray-400 min-w-[20px]',
    cellClassName: 'text-center border border-gray-300 text-xs',
    render: (_v, row) => (
      <>
        <div>{row.created_date ?? ''}</div>
        <div>{row.created_time ?? ''}</div>
      </>
    ),
  },
]

/** テーブル全体の Props */
export type HistoryTableProps<T extends BaseRow = BaseRow> = {
  rows: T[]
  columns: HistoryTableColumn<T>[]
  pressingIndex?: number | null
  sessionUserName?: string
  onPressStart?: (
    index: number,
    e: React.MouseEvent<HTMLTableRowElement> | React.TouchEvent<HTMLTableRowElement>
  ) => void
  onPressEnd?: (
    e: React.MouseEvent<HTMLTableRowElement> | React.TouchEvent<HTMLTableRowElement>
  ) => void
  touchStartY?: React.MutableRefObject<number | null>
  rowClassName?: (row: T, index: number) => string
  scrollContainerRef?: React.RefObject<HTMLDivElement | null>
  disableHover?: boolean
  getRowKey?: (row: T, index: number) => React.Key
  onRefresh?: () => void
  refreshing?: boolean
}

/** タッチスクロールとロングタップ判定用のしきい値(px) */
const SCROLL_THRESHOLD_PX = 8
/** 仮想スクロール用の推定行高さ（px) */
const ESTIMATED_ROW_HEIGHT = 40
/** 画面外に余分に描画しておく行数 */
const OVERSCAN_COUNT = 5

/** 列ごとのフィルタ状態 */
type FilterState = Record<string, Set<any>>

/** フィルタ対象とするヘッダー名（ホワイトリスト） */
const FILTERABLE_LABELS = new Set([
  // '日付',
  '時間',
  'カテゴリー',
  '開始時間',
  '終了時間',
  'クレーム内容',
  'サービス',
  'サービス内容',
])

/** テキスト位置のユーティリティ */
const alignClass = (align?: 'left' | 'center' | 'right') => {
  switch (align) {
    case 'center':
      return 'text-center'
    case 'right':
      return 'text-right'
    default:
      return 'text-left'
  }
}

/**
 * headerClassName から w-[..] / min-w-[..] を抽出して colgroup に適用する。
 * - table-layout: fixed の場合、col の width が最優先され、内容で幅が変わらない。
 * - min-w しかない列は、同値を width として扱い「旧コードの見え方」を維持する。
 */
const extractFixedWidth = (headerClassName?: string): string | undefined => {
  if (!headerClassName) return undefined
  const w = headerClassName.match(/(?:^|\s)w-\[([^\]]+)\]/)?.[1]
  if (w) return w
  const minW = headerClassName.match(/(?:^|\s)min-w-\[([^\]]+)\]/)?.[1]
  if (minW) return minW
  return undefined
}

function HistoryTableInner<T extends BaseRow = BaseRow>({
  rows,
  columns,
  pressingIndex,
  sessionUserName,
  onPressStart,
  onPressEnd,
  touchStartY,
  rowClassName,
  scrollContainerRef,
  disableHover = false,
  getRowKey,
  onRefresh,
  refreshing = false,
}: HistoryTableProps<T>) {
  /** タッチスクロール中かどうかの判定用 */
  const wasScrollingRef = useRef(false)

  /** スクロール位置の保存・復元用 */
  const prevScrollTopRef = useRef(0)

  /** ロングタップ判定用の開始位置（Y座標） */
  const internalTouchStartY = useRef<number | null>(null)
  const touchStartYRef = touchStartY ?? internalTouchStartY

  /** プルダウンリフレッシュ用の開始位置と状態 */
  const pullStartY = useRef<number | null>(null)
  const isPulling = useRef(false)

  /** 仮想スクロール用のスクロール位置と高さ */
  const [scrollTop, setScrollTop] = useState(0)
  const [viewportHeight, setViewportHeight] = useState(0)

  /** 列ごとのユニーク値一覧（文字列は前後の空白を除いて集計） */
  const uniqueValuesByColumn = useMemo(() => {
    const map: Record<string, any[]> = {}
    columns.forEach(col => {
      const key = col.key as string
      const values = rows.map(r => {
        const raw = r?.[key]
        if (typeof raw === 'string') return raw.trim()
        return raw
      })
      map[key] = Array.from(new Set(values))
    })
    return map
  }, [rows, columns])

  /** フィルタ状態と、どの列のフィルタモーダルが開いているか */
  const [filters, setFilters] = useState<FilterState>({})
  const [openFilterKey, setOpenFilterKey] = useState<string | null>(null)

  /** フィルタ適用後の行リスト（文字列は前後の空白を除いて比較） */
  const filteredRows = useMemo(() => {
    const activeKeys = Object.keys(filters)
    if (activeKeys.length === 0) return rows

    return rows.filter(row =>
      activeKeys.every(key => {
        const set = filters[key]
        if (!set) return true
        if (set.size === 0) return false

        const raw = row[key]
        const value = typeof raw === 'string' ? raw.trim() : raw
        return set.has(value)
      })
    )
  }, [rows, filters])

  /** 列ごとの値を全選択または全解除する処理 */
  const toggleFilterAll = useCallback(
    (field: string) => {
      setFilters(prev => {
        const allValues = uniqueValuesByColumn[field] ?? []
        if (allValues.length === 0) return prev

        const current = prev[field]
        const copy = { ...prev }

        // 何か一つでも選択されている → 一旦「全解除」
        if (!current || current.size > 0) {
          copy[field] = new Set()
          return copy
        }

        // current.size === 0 → 「全選択」
        const newFilters = { ...prev }
        delete newFilters[field]
        return newFilters
      })
    },
    [uniqueValuesByColumn]
  )

  /** 個別値のオン・オフ切り替え（文字列は trim して扱う） */
  const toggleFilterValue = useCallback(
    (field: string, value: any) => {
      hapticOn()
      setFilters(prev => {
        const allValues = uniqueValuesByColumn[field] ?? []
        if (allValues.length === 0) return prev

        const normalizedAllValues = allValues.map(v => (typeof v === 'string' ? v.trim() : v))
        const current = prev[field] ?? new Set(normalizedAllValues)
        const next = new Set(current)

        const normalizedValue = typeof value === 'string' ? value.trim() : value

        if (next.has(normalizedValue)) next.delete(normalizedValue)
        else next.add(normalizedValue)

        // 全ての値が選択されている状態になったらフィルタ自体を削除
        if (next.size === normalizedAllValues.length) {
          const copy = { ...prev }
          delete copy[field]
          return copy
        }

        return { ...prev, [field]: next }
      })
    },
    [uniqueValuesByColumn]
  )

  /** プルダウン開始位置を記録（スクロール位置が先頭のときのみ） */
  const handleGlobalTouchStart = (e: React.TouchEvent) => {
    if (scrollContainerRef?.current?.scrollTop === 0) {
      pullStartY.current = e.touches[0].clientY
      isPulling.current = false
    } else {
      pullStartY.current = null
    }
  }

  /** プルダウン距離を監視して、しきい値を超えたら「引いている状態」にする */
  const handleGlobalTouchMove = (e: React.TouchEvent) => {
    if (pullStartY.current !== null) {
      const offset = e.touches[0].clientY - pullStartY.current
      if (offset > refreshOffset) isPulling.current = true
    }
  }

  /** プルダウン終了時、しきい値を超えていればリフレッシュを実行 */
  const handleGlobalTouchEnd = () => {
    if (isPulling.current) onRefresh?.()
    pullStartY.current = null
    isPulling.current = false
  }

  /** スクロール位置の更新（仮想スクロール用） */
  const handleScroll: React.UIEventHandler<HTMLDivElement> = e => {
    const target = e.currentTarget
    setScrollTop(target.scrollTop)
    setViewportHeight(target.clientHeight)
  }

  /** フィルタモーダル表示中は縦横スクロールをロック */
  useEffect(() => {
    const el = scrollContainerRef?.current
    if (!el) return
    if (openFilterKey) el.style.overflow = 'hidden'
    else el.style.overflow = 'auto'
  }, [openFilterKey, scrollContainerRef])

  /** 初回レンダリング時にビューポート高さを取得 */
  useEffect(() => {
    if (scrollContainerRef?.current) {
      setViewportHeight(scrollContainerRef.current.clientHeight)
    }
  }, [scrollContainerRef])

  /** 行データが差し替わってもスクロール位置を維持する */
  useEffect(() => {
    if (scrollContainerRef?.current) {
      scrollContainerRef.current.scrollTop = prevScrollTopRef.current
    }
  }, [rows, scrollContainerRef])

  /** スクロール位置監視＋iOSダブルタップ拡大防止＋プルダウンリフレッシュのイベント登録 */
  useLayoutEffect(() => {
    const el = scrollContainerRef?.current
    if (!el) return

    setViewportHeight(el.clientHeight)

    const onScroll = () => {
      setScrollTop(el.scrollTop)
      setViewportHeight(el.clientHeight)
      prevScrollTopRef.current = el.scrollTop
    }

    // iOSのダブルタップ拡大防止（コンテナ限定）
    let lastTapTime = 0
    const preventDoubleTapZoom = (e: TouchEvent) => {
      const now = Date.now()
      if (now - lastTapTime < 300) e.preventDefault()
      lastTapTime = now
    }

    const onGlobalTouchStart = (e: TouchEvent) => {
      if (el.scrollTop === 0) {
        pullStartY.current = e.touches[0]?.clientY ?? null
        isPulling.current = false
      } else {
        pullStartY.current = null
      }
    }
    const onGlobalTouchMove = (e: TouchEvent) => {
      if (pullStartY.current !== null) {
        const offset = e.touches[0].clientY - pullStartY.current
        if (offset > refreshOffset) isPulling.current = true
      }
    }
    const onGlobalTouchEnd = () => {
      if (isPulling.current) onRefresh?.()
      pullStartY.current = null
      isPulling.current = false
    }

    el.addEventListener('scroll', onScroll)
    el.addEventListener('touchstart', preventDoubleTapZoom, { passive: false })
    el.addEventListener('touchstart', onGlobalTouchStart, { passive: true })
    el.addEventListener('touchmove', onGlobalTouchMove, { passive: true })
    el.addEventListener('touchend', onGlobalTouchEnd, { passive: true })

    return () => {
      el.removeEventListener('scroll', onScroll)
      el.removeEventListener('touchstart', preventDoubleTapZoom as any)
      el.removeEventListener('touchstart', onGlobalTouchStart as any)
      el.removeEventListener('touchmove', onGlobalTouchMove as any)
      el.removeEventListener('touchend', onGlobalTouchEnd as any)
    }
  }, [scrollContainerRef, onRefresh])

  /** 仮想スクロール用の各種計算値 */
  const totalCount = filteredRows.length
  const totalHeight = totalCount * ESTIMATED_ROW_HEIGHT

  const visibleCount =
    viewportHeight > 0
      ? Math.ceil(viewportHeight / ESTIMATED_ROW_HEIGHT) + OVERSCAN_COUNT * 2
      : totalCount

  const startIndex = Math.max(0, Math.floor(scrollTop / ESTIMATED_ROW_HEIGHT) - OVERSCAN_COUNT)
  const endIndex = Math.min(totalCount, startIndex + visibleCount)

  const visibleRows = filteredRows.slice(startIndex, endIndex)
  const topPaddingHeight = startIndex * ESTIMATED_ROW_HEIGHT
  const bottomPaddingHeight =
    totalHeight - topPaddingHeight - visibleRows.length * ESTIMATED_ROW_HEIGHT

  // フィルタモダール下にスクロールする時、動作しないよう
  const touchTargetRef = useRef<number | null>(null)
  const touchMovedRef = useRef(false)

  /** colgroup 用: 各列の固定 width を抽出 */
  const colWidths = useMemo(() => {
    return columns.map(col => extractFixedWidth(col.headerClassName))
  }, [columns])

  return (
    <div
      ref={scrollContainerRef}
      onTouchStart={handleGlobalTouchStart}
      onTouchMove={handleGlobalTouchMove}
      onTouchEnd={handleGlobalTouchEnd}
      onScroll={handleScroll}
      className='w-full'
      style={{
        touchAction: 'manipulation',
        overflowY: openFilterKey ? 'hidden' : 'auto',
        // ここがポイント：固定幅の列合計が画面を超える場合は横スクロールで逃がす
        overflowX: openFilterKey ? 'hidden' : 'auto',
        height: '52vh',
      }}
    >
      {/* リフレッシュ中のインジケータ */}
      {refreshing && (
        <div className='sticky top-0 flex items-center justify-center bg-transparent py-2'>
          <span className='inline-block h-4 w-4 animate-spin rounded-full border-2 border-gray-300 border-t-gray-600' />
          <span className='ml-2 text-sm text-gray-500'>{COMMON_MESSAGES.LOADING_DATA}</span>
        </div>
      )}

      {/* 件数表示とフィルタ解除ボタン */}
      <div className='sticky top-0 left-0 flex items-center justify-between bg-white px-2 pt-1 pb-1 text-xs font-semibold text-gray-700'>
        {Object.keys(filters).length > 0 ? (
          <button
            type='button'
            className='rounded-[6px] border border-blue-300 bg-blue-100 px-2 py-0.5 text-xs font-semibold text-cyan-700 hover:bg-blue-200'
            onClick={() => {
              hapticOn()
              setFilters({})
              setOpenFilterKey(null)
            }}
          >
            全フィルタ解除
          </button>
        ) : (
          <span />
        )}
        <span>
          データ件数：<span className='text-sm font-bold text-blue-700'>{filteredRows.length}</span>件
        </span>
      </div>

      {/* table-fixed + colgroup で列幅を完全固定（内容で幅が変わらない） */}
      <table
        className='w-full table-fixed border-collapse select-none'
        style={{
          userSelect: 'none',
          WebkitUserSelect: 'none',
        }}
      >
        <colgroup>
          {colWidths.map((w, idx) => (
            <col
              key={idx}
              style={
                w
                  ? {
                      width: w,
                    }
                  : undefined
              }
            />
          ))}
        </colgroup>

        <thead
          style={{
            position: 'sticky',
            top: 24,
            background: '#558ED5',
          }}
        >
          <tr className='bg-[#558ED5] font-bold text-white'>
            {columns.map(col => {
              const fieldKey = col.key as string
              const uniqueValues = uniqueValuesByColumn[fieldKey] ?? []

              const rawSet = filters[fieldKey]
              const selectedSet = rawSet ?? new Set(uniqueValues)

              const allSelected =
                selectedSet.size === uniqueValues.length && uniqueValues.length > 0
              const noneSelected = rawSet !== undefined && rawSet.size === 0

              const isOpen = openFilterKey === fieldKey
              const isFilterable = FILTERABLE_LABELS.has(col.label)
              const isFiltered = filters[fieldKey] !== undefined

              return (
                <th
                  key={col.key}
                  scope='col'
                  className={`${alignClass(col.align)} ${col.headerClassName ?? ''} ${
                    isFiltered ? 'bg-[#3a6fb5]' : ''
                  } overflow-hidden text-ellipsis whitespace-nowrap`}
                >
                  {isFilterable ? (
                    <>
                      <div className='flex items-center justify-center gap-1 px-1 text-xs whitespace-nowrap'>
                        <span className='max-w-[80%] truncate'>{col.label}</span>
                        <button
                          type='button'
                          onClick={e => {
                            hapticOn()
                            e.stopPropagation()
                            setOpenFilterKey(prev => (prev === fieldKey ? null : fieldKey))
                          }}
                          className={`shrink-0 rounded border border-white/60 px-1 py-[1px] text-[10px] ${
                            isFiltered
                              ? 'bg-black/10 hover:bg-black/20'
                              : 'bg-white/10 hover:bg-white/20'
                          }`}
                        >
                          <span
                            className='material-symbols-outlined text-xs leading-none'
                            style={{ fontSize: '11px', lineHeight: 2 }}
                          >
                            {isFiltered ? 'filter_alt' : 'expand_more'}
                          </span>
                        </button>
                      </div>

                      {isOpen && (
                        <div
                          className='fixed inset-1 z-50 mt-50 flex items-center justify-center bg-black/35'
                          onClick={() => setOpenFilterKey(null)}
                        >
                          <div
                            className='flex max-h-[75vh] w-[100vw] max-w-sm flex-col rounded-md border border-gray-300 bg-white text-gray-800 shadow-lg'
                            style={{ overflow: 'hidden' }}
                            onClick={e => e.stopPropagation()}
                          >
                            <div className='border-b border-gray-200 px-3 py-2 text-xs font-semibold'>
                              {col.label}を絞り込む
                            </div>

                            <div
                              className='flex cursor-pointer items-center gap-2 border-b border-gray-100 px-3 py-2 text-xs'
                              onClick={e => {
                                hapticOn()
                                e.stopPropagation()
                                e.preventDefault()
                                toggleFilterAll(fieldKey)
                              }}
                            >
                              <span className='material-symbols-outlined text-lg text-blue-600'>
                                {allSelected ? 'check_box' : 'check_box_outline_blank'}
                              </span>
                              <span className='text-sm text-gray-800'>
                                {noneSelected ? '全て選択' : '全て解除'}
                              </span>
                            </div>

                            <div className='max-h-[27vh] flex-1 overflow-y-auto px-3 py-2'>
                              {uniqueValues.length === 0 ? (
                                <div className='text-[11px] text-gray-500'>データがありません</div>
                              ) : (
                                uniqueValues.map((v, idx) => {
                                  const label = v == null ? '(空白)' : String(v)
                                  const checked = selectedSet.has(v)

                                  return (
                                    <div
                                      key={`${label}-${idx}`}
                                      className='flex cursor-pointer items-center gap-3 rounded px-2 py-2 hover:bg-gray-100 active:bg-gray-200'
                                      onClick={e => {
                                        hapticOn()
                                        e.preventDefault()
                                        e.stopPropagation()
                                        toggleFilterValue(fieldKey, v)
                                      }}
                                      onTouchStart={() => {
                                        touchTargetRef.current = idx
                                        touchMovedRef.current = false
                                      }}
                                      onTouchMove={() => {
                                        touchMovedRef.current = true
                                      }}
                                      onTouchEnd={e => {
                                        e.preventDefault()
                                        e.stopPropagation()
                                        if (touchTargetRef.current === idx && !touchMovedRef.current) {
                                          toggleFilterValue(fieldKey, v)
                                        }
                                        touchTargetRef.current = null
                                        touchMovedRef.current = false
                                      }}
                                    >
                                      <input
                                        type='checkbox'
                                        readOnly
                                        checked={checked}
                                        onClick={e => {
                                          e.stopPropagation()
                                          e.preventDefault()
                                          toggleFilterValue(fieldKey, v)
                                        }}
                                        className='h-5 w-5'
                                      />
                                      <span className='text-xs break-all text-gray-800' title={label}>
                                        {label}
                                      </span>
                                    </div>
                                  )
                                })
                              )}
                            </div>

                            <div className='flex justify-end border-t border-gray-100 px-3 py-2'>
                              <button
                                type='button'
                                className='rounded border border-gray-300 bg-gray-50 px-10 py-1 text-xs hover:bg-gray-100'
                                onClick={() => {
                                  hapticOn()
                                  setOpenFilterKey(null)
                                }}
                              >
                                閉じる
                              </button>
                            </div>
                          </div>
                        </div>
                      )}
                    </>
                  ) : (
                    <div className='px-1 text-xs whitespace-nowrap overflow-hidden text-ellipsis'>
                      <span className='max-w-[80%] truncate'>{col.label}</span>
                    </div>
                  )}
                </th>
              )
            })}
          </tr>
        </thead>

        <tbody>
          {/* 上側のダミー行（仮想スクロール用） */}
          {topPaddingHeight > 0 && (
            <tr>
              <td colSpan={columns.length} style={{ height: topPaddingHeight }} />
            </tr>
          )}

          {/* 実際に表示する範囲のみを描画 */}
          {visibleRows.map((row, localIndex) => {
            const globalIndex = startIndex + localIndex
            const originalIndex = rows.indexOf(row)

            const baseClass =
              pressingIndex === originalIndex
                ? 'bg-gray-100'
                : row.creator === sessionUserName
                  ? 'bg-[#FFFFD5]'
                  : disableHover
                    ? ''
                    : 'hover:bg-gray-100'

            const customClass = rowClassName ? rowClassName(row, globalIndex) : ''
            const key = getRowKey?.(row, globalIndex) ?? globalIndex

            return (
              <tr
                key={key}
                className={`${baseClass} ${customClass} no-hover`}
                onMouseDown={e => {
                  touchStartYRef.current = e.clientY
                  wasScrollingRef.current = false
                  if (scrollContainerRef?.current) {
                    prevScrollTopRef.current = scrollContainerRef.current.scrollTop
                  }
                  onPressStart?.(originalIndex, e)
                }}
                onMouseUp={e => {
                  if (wasScrollingRef.current || touchStartYRef.current === null) return
                  onPressEnd?.(e)
                }}
                onMouseMove={e => {
                  const startY = touchStartYRef.current
                  const currentY = e.clientY
                  if (
                    startY !== null &&
                    currentY !== null &&
                    Math.abs(currentY - startY) > SCROLL_THRESHOLD_PX
                  ) {
                    wasScrollingRef.current = true
                    onPressEnd?.(e)
                    touchStartYRef.current = null
                  }
                }}
                onTouchStart={e => {
                  touchStartYRef.current = e.touches[0]?.clientY ?? null
                  wasScrollingRef.current = false
                  if (scrollContainerRef?.current) {
                    prevScrollTopRef.current = scrollContainerRef.current.scrollTop
                  }
                  onPressStart?.(originalIndex, e)
                }}
                onTouchMove={e => {
                  const startY = touchStartYRef.current
                  const currentY = e.touches[0]?.clientY ?? null
                  if (
                    startY !== null &&
                    currentY !== null &&
                    Math.abs(currentY - startY) > SCROLL_THRESHOLD_PX
                  ) {
                    wasScrollingRef.current = true
                    onPressEnd?.(e)
                    touchStartYRef.current = null
                  }
                }}
                onTouchEnd={e => {
                  if (wasScrollingRef.current || touchStartYRef.current === null) return
                  onPressEnd?.(e)
                }}
              >
                {columns.map(col => {
                  const value = row[col.key]
                  return (
                    <td
                      key={String(col.key)}
                      className={`${alignClass(col.align)} ${col.cellClassName ?? ''} overflow-hidden text-ellipsis whitespace-nowrap`}
                    >
                      {col.render ? col.render(value, row) : value}
                    </td>
                  )
                })}
              </tr>
            )
          })}

          {/* 下側のダミー行（仮想スクロール用） */}
          {bottomPaddingHeight > 40 && (
            <tr>
              <td colSpan={columns.length} style={{ height: bottomPaddingHeight }} />
            </tr>
          )}

          {/* 行が一件もない場合の表示 */}
          {totalCount === 0 && (
            <tr>
              <td colSpan={columns.length} className='py-4 text-center text-sm text-gray-500'>
                データがありません
              </td>
            </tr>
          )}
        </tbody>
      </table>
    </div>
  )
}

const HistoryTable = React.memo(HistoryTableInner) as typeof HistoryTableInner
export default HistoryTable

```
