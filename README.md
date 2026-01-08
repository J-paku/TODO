```
import axios, { AxiosError, AxiosResponse } from 'axios'
import { useCallback, useMemo } from 'react'
import { default as modelApi } from '@/api/model'
import { default as imageApi } from '@/api/image'
import { default as touenApi } from '@/api/touen'
import { default as errorApi } from '@/api/error'
import useToast from '@/hooks/useToast'
// import { handleApiError } from "@/lib/fetchSteptask"; //for e.g.
export type HttpRequest = (
  requestFunction: () => Promise<AxiosResponse>,
  disableDefaultError?: boolean
) => Promise<AxiosResponse | AxiosError | undefined>

export default function useHttp() {
  const { openToast } = useToast()
  const httpRequest: HttpRequest = useCallback(
    async (apiFunction: () => Promise<AxiosResponse>, disableDefaultError = false) => {
      try {
        const response: AxiosResponse = await apiFunction()
        return response
      } catch (error) {
        if (axios.isAxiosError(error) && error.message) {
          if (disableDefaultError) return error
          if (error.response) {
            ///do something e.g. for handler case error
            // switch (error.code) {
            //   case xxxx:
            //     break;

            //   default:
            //     openToast({
            //       message: `APIリクエスト失敗:${error.code}`,
            //       position: "center",
            //     });
            //     break;
            // }
            return error
          } else {
            // error without code and response
            openToast.error(
              `ネットワークが不安定の為、更新できませんでした。（APIリクエスト失敗:${error.message}）`,
              'center'
            )
            // can be call exist function or create this here
            // handleApiError(error)
            return error
          }
        }
      }
    },
    [openToast]
  )
  //  api group by result or purpose
  const api = useMemo(() => {
    return {
      model: modelApi(httpRequest),
      image: imageApi(httpRequest),
      touen: touenApi(httpRequest),
      error: errorApi(httpRequest),
      //e.g. product : productApi(httpRequest)
      //under productApi must have product Api only
    }
  }, [httpRequest])
  return { api }
}

```

```
import axios, { AxiosInstance } from 'axios'
import { z } from 'zod'

export type ApiResponse<T> = {
  pagination?: {
    Offset: number
    PageSize: number
    TotalCount: number
  }
  data: T | null | undefined
  message: string
  code: number | string
  _error?: z.core.$ZodIssue[]
}

// this const is load on App start should be careful
export const axiosInstance: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_PLEASANTER_URL, //base url config here or env
  // headers: {
  //   Accept: '*/*', // for test
  //   'Access-Control-Allow-Origin': '*', // for test
  //   'Content-Type': 'application/json',
  // },
})
// Request Interceptor
axiosInstance.interceptors.request.use(
  config => {
    config.params = {
      ...config.params,
      // version: '1.0'
    }
    if (config.method === 'post' || config.method === 'put') {
      config.data = {
        ...config.data,
        ...(process.env.NODE_ENV === 'development' && {
          Apikey: process.env.NEXT_PUBLIC_API_PLEASANTER_API_KEY,
        }),
        ApiVersion: '1.1',
      }
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)

// Response Interceptor
axiosInstance.interceptors.response.use(
  response => {
    return response
  },
  async error => {
    if (error.response && error.response.data) {
      const contentType = error.response.header['content-type']
      if (contentType && contentType.includes('application/json')) {
        try {
          const text = await error.response.data.text() // Convert Blob to Text for handle error
          const json = JSON.parse(text)
          error.response.data = json
        } catch (err) {
          console.error('Failed to parse JSON :', err)
        }
      }
    }
    return Promise.reject(error)
  }
)

```
