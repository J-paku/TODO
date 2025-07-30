const reprintEventBtn = (
  clientName: string,
  updatedTime: string,
  historyItems: { item: string; qty: number; }[]
) => {
  console.log("重要！(SWR clients 사용)");
  console.log("updatedTime:", updatedTime);
  console.log("clientName:", clientName);

  const kaiGyou = "\n";
  const kaiGyouTwo = " \n \n \n ";

  const title = "納品書 (KMS)" + " \n ";
  const dateTime = (centerAlign(updatedTime, 45) + " \n ").replace("T", " ");
  const clientLine = `${clientName} 様`;
  const productLine = "--------------------------------";

  // 🔁 SWR에서 최신 데이터 사용
  const groupedItems = clients.filter(
    item =>
      item.created_date + 'T' + item.created_time === updatedTime &&
      item.clients_name === clientName
  );

  if (groupedItems.length === 0) {
    console.warn("❗ groupedItemsが空です。データが一致しないか、まだ取得されていない可能性があります。");
    console.log("SWR clients sample:", clients.slice(0, 3));
  }

  try {
    const itemLines = groupedItems.length === 0
      ? [``]
      : groupedItems.map(item => {
          const name = hankakuToZenkakuKatakana(item?.item || "").padEnd(14, '　');
          const qty = String(item.qty);
          return `${name} :  ${qty}  ${kaiGyou} ${kaiGyou}`;
        });

    console.log("itemLines:", itemLines);
    console.log("itemLines raw:", JSON.stringify(itemLines));

    const printDescription = [
      title,
      kaiGyou,
      dateTime,
      clientLine,
      kaiGyou,
      productLine,
      ...itemLines,
      productLine,
      kaiGyouTwo,
    ].join("\n");

    console.log("printDescription1:", printDescription);
    console.log("printDescription2:", JSON.stringify(printDescription));

    const win = window as any;

    let retryInterval: NodeJS.Timeout;
    let startTime = Date.now();
    let retryCount = 0;
    const maxRetries = 18 + 1;

    const tryPrint = () => {
      retryCount++;
      const remaining = maxRetries - retryCount;

      if (Date.now() - startTime > 90_000) {
        hapticOn("error");
        clearInterval(retryInterval);
        setSecond(3000);
        setPrintStatus("🖨️を見つかりませんでした。");
        setLoading(false);
        setShowToast(true);
        return;
      }

      if (win.webkit?.messageHandlers?.printHandler) {
        win.webkit.messageHandlers.printHandler.postMessage(printDescription);
        hapticOn("medium");
        setSecond(4700);
        setPrintStatus(`${process.env.NEXT_PUBLIC_PRINT_WAIT_MESSAGE} ${remaining}回`);
      } else {
        hapticOn("error");
        clearInterval(retryInterval);
        setSecond(3000);
        setPrintStatus("印刷失敗。iOS端末を確認してください。");
        setLoading(false);
        setShowToast(true);
      }
    };

    setLoading(true);

    tryPrint();
    retryInterval = setInterval(tryPrint, 5000);

    window.onPrintResult = function (message: string) {
      if (message.includes("✅")) {
        hapticOn("success");
        clearInterval(retryInterval);
        setPrintStatus("🖨️ 再印刷成功しました");
        setLoading(false);
        setShowToast(true);
      } else {
        const remaining = maxRetries - retryCount;
        setSecond(4800);
        setPrintStatus(`${process.env.NEXT_PUBLIC_PRINT_WAIT_MESSAGE} ${remaining}回`);
        setShowToast(true);
      }
    };
  } catch (e) {
    console.error("エラー:", e);
  }
};