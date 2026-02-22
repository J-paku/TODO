아래는 이 프로젝트의 `IndexedDB` 사용법과 구조 요약이야. 필요하면 특정 함수 사용 예시까지 추가해줄게.

**구조**
- DB 이름: `PipitAppDB` (운영) / `PipitAppDB_TEST` (그 외)
- ObjectStore: `Favorites` 하나만 사용
- 스키마 버전: `SCHEMA_VERSION = 4`
- DB 연결은 싱글톤(`dbPromise`)으로 관리

**초기화/오픈 흐름**
1. `canUseIndexedDB()`로 환경 체크
2. `readExistingVersion()`로 현재 버전 확인(타임아웃 포함)
3. `openWithVersion()`에서:
   - `onupgradeneeded` 시 스키마 생성
   - `onblocked` 또는 특정 에러 시 DB 삭제 후 재오픈 시도
4. 외부에서는 `getDB()`만 사용 (실패 시 `null` 반환)

**저장/조회 방식**
- 메뉴 순서/임의 객체는 JSON → `lz-string`으로 UTF-16 압축 저장
- 즐겨찾기는 plain string 저장
- 조회 시:
  - JSON 문자열이면 그대로 파싱
  - 압축 문자열이면 `decompressFromUTF16` 후 파싱

**주요 API**
- 메뉴 순서(압축 저장)
  - `setMenuOrder(key, order: string[])`
  - `getMenuOrder(key)`
- 임의 객체(압축 저장)
  - `setMenuOrderObject<T>(key, data)`
  - `getMenuOrderObject<T>(key)`
  - `deleteMenuOrderObject(key)`
- 즐겨찾기(plain string)
  - `setFavorite(key, value)`
  - `getFavorite(key)`
  - `removeFavorite(key)`

**캐시 폴백**
- 사용자 정보 캐시는 IndexedDB 실패 시 `sessionStorage`로 폴백:
  - `setUserInfoCache<T>(value)`
  - `getUserInfoCache<T>()`
  - `removeUserInfoCache()`

**진단**
- `probeIndexedDBRoundTrip()`로 IndexedDB 왕복 테스트
  - 실패 시 `sessionStorage`로 대체 검사

원하면 “실제 호출 예시(React 훅/컴포넌트)” 형태로 정리해줄게.
