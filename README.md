import { waitForIndexedDBReady } from "../utils/waitForIndexedDBReady";

export function useLaunchFavorite(safePush, setIsNavigating) {
  const { getFavorite } = useFavoriteLS();

  useEffect(() => {
    const launchFavorite = async () => {
      const favoriteId = getFavorite();
      const hasLaunched = sessionStorage.getItem("appLaunched");
      const justSet = localStorage.getItem("favoriteJustSet") === "true";
      if (justSet) {
        localStorage.removeItem("favoriteJustSet");
        return;
      }

      if (!hasLaunched && favoriteId) {
        // ✅ Safari의 IndexedDB 초기화를 기다림
        await waitForIndexedDBReady();

        setIsNavigating(true);
        try {
          const targetPath = paths[favoriteId];
          if (!targetPath) return;

          const { latitude, longitude } = await ensureCoordsInLocalStorage();
          const data = await getClaimClients(latitude.toString(), longitude.toString());

          await Promise.all([
            setMenuOrderObject("クレーム：得意先", data.allItems ?? []),
            setMenuOrderObject("クレーム：最寄り得意先", Array.isArray(data.closest) ? data.closest : []),
          ]);

          safePush(`${targetPath}?nc=${Date.now().toString().slice(-2)}`);
        } catch (e) {
          console.error("お気に入り自動移動エラー:", e);
        } finally {
          setTimeout(() => setIsNavigating(false), 1500);
        }
      }
    };
    launchFavorite();
  }, [getFavorite, safePush, setIsNavigating]);
}
// utils/waitForIndexedDBReady.ts
export async function waitForIndexedDBReady(timeout = 800) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    try {
      const test = indexedDB.open("test");
      return; // 성공적으로 열렸다면 준비 완료
    } catch {
      await new Promise((r) => setTimeout(r, 50));
    }
  }
  console.warn("IndexedDB init timeout — Safari fallback may apply");
}


