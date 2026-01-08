```
// api/touen/count/getSteptaskSyouhin.ts
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'

export type PayloadSteptaskSyouhin = {
  ApiVersion: number
  Offset: number
  PageSize: number
  View: {
    ColumnFilterHash: {
      Title: string
    }
    ColumnFilterSearchTypes: {
      Title: 'ExactMatch' | string
    }
  }
}

// 응답의 Data 전체를 반환하도록(가공은 훅에서)
export default async function getSteptaskSyouhin(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<any>> {
  const response = await httpRequest(() =>
    axiosInstance.post(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return {
      code: response.code ?? 500,
      message: response.message,
      data: undefined,
      _error: response,
    }
  }

  return {
    code: 200,
    message: 'api get successed',
    data: response?.data, // ← Response 전체 반환
  }
}

```

```
// api/touen/index.ts (사용자 코드 기반 수정)
import getSteptaskSyouhin, { PayloadSteptaskSyouhin } from './count/getSteptaskSyouhin'

// ...
interface TouenApi {
  // ...
  getSteptaskSyouhin: (payload: PayloadSteptaskSyouhin) => Promise<ApiResponse<any>>
  // ...
}

export default function touen(httpRequest: HttpRequest): TouenApi {
  return {
    // ...
    getSteptaskSyouhin: payload => getSteptaskSyouhin(httpRequest, payload),
    // ...
  }
}
```

```
// hooks/useTouenCount.ts (해당 부분만 수정)
import { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

const getSteptaskSyouhin = useMemo(
  () => async (payload: PayloadSteptaskSyouhin) => {
    const response = await api.touen.getSteptaskSyouhin(payload)
    if (response.code !== 200) {
      // 필요 시 에러 처리 정책 적용 (toast/log/boundary)
      openToast.error(response.message, 'center')
      return undefined
    }
    return response.data
  },
  [api, openToast]
)

```

```
// hooks/useFetchTouenItems.ts (예시: 필요한 함수만 발췌)
import { useCallback } from 'react'
import useTouenCount from '@/hooks/useTouenCount'
import type { PayloadSteptaskSyouhin } from '@/api/touen/count/getSteptaskSyouhin'

type StepTaskResponse = {
  Response?: {
    TotalCount?: number
    Data?: any[]
  }
}

export function useFetchTouenItemsCore() {
  const { getSteptaskSyouhin } = useTouenCount()

  const fetchAllSteptaskSyouhin = useCallback(
    async (tokuisaki: string) => {
      const pageSize = 200

      // 1) TotalCount 取得用 payload
      const initialPayload: PayloadSteptaskSyouhin = {
        ApiVersion: 1.1,
        Offset: 0,
        PageSize: 1,
        View: {
          ColumnFilterHash: { Title: tokuisaki },
          ColumnFilterSearchTypes: { Title: 'ExactMatch' },
        },
      }

      const initialRaw = (await getSteptaskSyouhin(initialPayload)) as StepTaskResponse | undefined
      const totalCount = initialRaw?.Response?.TotalCount

      if (typeof totalCount !== 'number') {
        throw new Error('TotalCountが不正です')
      }

      // 2) 페이지 병렬 요청
      const reqs: Promise<any>[] = []
      for (let offset = 0; offset < totalCount; offset += pageSize) {
        const payload: PayloadSteptaskSyouhin = {
          ...initialPayload,
          Offset: offset,
          PageSize: pageSize,
        }
        reqs.push(getSteptaskSyouhin(payload))
      }

      const pages = (await Promise.all(reqs)) as StepTaskResponse[]
      const list = pages.flatMap(p => p?.Response?.Data ?? [])

      return list
    },
    [getSteptaskSyouhin]
  )

  return { fetchAllSteptaskSyouhin }
}
```
