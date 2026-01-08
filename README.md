```
// 필요한 최소 타입(프로젝트에 이미 타입 있으면 그걸로 대체)
type SteptaskItemsGetResponse<T = unknown> = {
  Response?: {
    TotalCount?: number
    Data?: T[]
  }
}

export default async function getSteptaskSyouhin<T = unknown>(
  httpRequest: HttpRequest,
  payload: PayloadSteptaskSyouhin
): Promise<ApiResponse<SteptaskItemsGetResponse<T>>> {
  const response = await httpRequest(() =>
    axiosInstance.post(TOUEN_API_ENDPOINTS.ITEMS_GET, payload)
  )

  if (axios.isAxiosError(response)) {
    return { code: response.code ?? 500, message: response.message, data: undefined }
  }

  return {
    code: 200,
    message: 'api get succeeded',
    data: response.data, // 여기서 "전체"를 넘김
  }
}

```
