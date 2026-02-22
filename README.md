# indexedDB.ts 요약

## 개요
- IndexedDB 사용 유틸리티 모듈.
- `PipitAppDB`(운영) / `PipitAppDB_TEST`(그 외)로 DB명을 분기.
- 단일 ObjectStore `Favorites` 사용.
- 스키마 최소 버전 `SCHEMA_VERSION = 4`.

## 핵심 흐름
- `canUseIndexedDB()`로 사용 가능 여부의 1차 체크.
- `readExistingVersion()`으로 현재 DB 버전 읽기(타임아웃 포함).
- `openWithVersion()`에서 `onupgradeneeded`로 스키마 생성, `onblocked`/에러 시 자동 복구(삭제 후 재시도).
- `getDBUnsafe()`는 예외를 그대로 throw, `getDB()`는 실패 시 `null` 반환(안전 래퍼).
- `dbPromise`로 DB 연결을 싱글톤 관리하며 실패 시 초기화.

## 저장/조회 API
- 공통: 모두 `getDB()` 사용, 실패 시 경고 로그만 남기고 조용히 종료.
- `setMenuOrder(key, order: string[])`
  - JSON 직렬화 후 `lz-string`으로 UTF-16 압축해 저장.
- `getMenuOrder(key)`
  - 저장값이 JSON 문자열인지 확인 후 복원.
- `setFavorite(key, value: string)`
- `getFavorite(key)`
- `removeFavorite(key)`
- `setMenuOrderObject<T>(key, data: T)`
  - 임의 객체를 JSON + 압축 저장.
- `getMenuOrderObject<T>(key)`
  - JSON 문자열이면 그대로 파싱, 아니면 압축 해제 후 파싱.
- `deleteMenuOrderObject(key)`

## 사용자 정보 캐시
- `setUserInfoCache<T>(value)`
  - IndexedDB 저장 실패 시 `sessionStorage`로 폴백.
- `getUserInfoCache<T>()`
  - IndexedDB → `sessionStorage` 순서로 조회.
- `removeUserInfoCache()`
  - IndexedDB 삭제 실패 시 `sessionStorage` 삭제 시도.

## 진단/복구
- `probeIndexedDBRoundTrip()`
  - IndexedDB 왕복 저장/조회 가능 여부 확인.
  - 실패 시 `sessionStorage`로 대체 검사.
- `resetDbSingletonForDebug()`
  - 싱글톤 DB 연결 강제 초기화.

## 주의 포인트
- `onblocked` 발생 시 자동 DB 삭제 후 재오픈을 시도(특히 Safari/WKWebView 대응).
- 실패 시 앱 전체가 죽지 않도록 대부분 안전하게 `null`/경고 로그로 처리.
