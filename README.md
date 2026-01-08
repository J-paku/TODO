```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse, axiosInstance } from '@/api'
import axios from 'axios'
import { TOUEN_API_ENDPOINTS } from '@/constants/api/touen'
export type PayloadSteptaskSyouhin = {
  View: {
    PageSize: number
    Offset: number
    ColumnFilterHash: {
      Title: string
    }
    ColumnFilterSearchTypes: {
      Title: string
    }
  }
}

export default async function getSteptaskSyouhin(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<string>> {
  const response = await httpRequest(() =>
    axiosInstance.post(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )
  if (axios.isAxiosError(response)) {
    return {
      code: response.code ?? 500,
      message: response.message,
      data: undefined,
    }
  }
  return {
    code: 200,
    message: 'api get successed', //something wording
    data: response?.data.Response.Data[0].仕入パック単価,
  }
}
```

```
import { HttpRequest } from '@/hooks/useHttp'
import { ApiResponse } from '@/api'
import getTouenList, { PayloadGetTouenList } from './list/getTouenList'
import createTouenCount, { PayloadCreateTouenCount } from './count/createTouenCount'
import getTouenClientId from './count/getTouenClientId'
import getSelectedTouenPrice, { PayloadSelectedTouenPrice } from './count/getSelectedTouenPrice'
import deleteSelectedTouen from './list/deleteSelectedTouen'
import { FilterTouenData, FilterTouenDataNotIn, TouenList } from './count/types'
import getTouenStepTaskDataIn, {
  PayloadGetTouenStepTaskDataIn,
} from './list/getTouenStepTaskDataIn'
import getTouenStepTaskDataNotIn, {
  PayloadGetTouenStepTaskDataNotIn,
} from './list/getTouenStepTaskDataNotIn'

interface TouenApi {
  // 登園カウント
  createTouenCount: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>
  getTouenClientId: (resultId: string) => Promise<ApiResponse<string>>
  getSelectedTouenPrice: (payload: PayloadSelectedTouenPrice) => Promise<ApiResponse<string>>
  getSteptaskSyouhin: (payload: PayloadCreateTouenCount) => Promise<ApiResponse<string>>

  // 登園リスト
  getTouenList: (
    payload: PayloadGetTouenList<FilterTouenData | FilterTouenDataNotIn>
  ) => Promise<ApiResponse<TouenList[]>>
  getTouenStepTaskDataIn: (payload: PayloadGetTouenStepTaskDataIn) => Promise<ApiResponse<string>>
  getTouenStepTaskDataNotIn: (
    payload: PayloadGetTouenStepTaskDataNotIn
  ) => Promise<ApiResponse<string>>
  deleteSelectedTouen: (result: string) => Promise<ApiResponse<string>>
}

export default function touen(httpRequest: HttpRequest): TouenApi {
  return {
    // 登園カウント
    createTouenCount: payload => createTouenCount(httpRequest, payload),
    getTouenClientId: resultId => getTouenClientId(httpRequest, resultId),
    getSelectedTouenPrice: payload => getSelectedTouenPrice(httpRequest, payload),

    // 登園リスト
    getTouenList: payload => getTouenList(httpRequest, payload),
    getTouenStepTaskDataIn: payload => getTouenStepTaskDataIn(httpRequest, payload),
    getTouenStepTaskDataNotIn: payload => getTouenStepTaskDataNotIn(httpRequest, payload),
    deleteSelectedTouen: resultId => deleteSelectedTouen(httpRequest, resultId),
  }
}
```
