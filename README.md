<style>
  @page {
    size: 43mm 80mm;
    margin: 0;
  }
  body {
    width: 43mm;
    height: 80mm;
    margin: 0;
    padding: 0;
  }
</style>

const htmlContent = `
  <html>
    <head>
      <style>
        @page { size: 43mm 80mm; margin: 0; }
        body { width: 43mm; height: 80mm; margin: 0; padding: 0; }
        .label { font-size: 12pt; padding: 5mm; }
      </style>
    </head>
    <body>
      <div class="label">
        <div>得意先名: ${selectedItem.title}</div>
        <div>住所: ${selectedItem.住所}</div>
        <div>連絡先: ${selectedItem.連絡先}</div>
      </div>
    </body>
  </html>
`;

const { uri } = await Print.printToFileAsync({ html: htmlContent });
await Sharing.shareAsync(uri);

	•	react-native-thermal-receipt-printer 같은 라이브러리 이용하여 직접 프린터 제어.
	•	**ZPL (Zebra Printer Language)**로 라벨 작성 → 프린터 전송