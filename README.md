```
func tryPrint(text: String) {
        let bold: CharacterBold = .SII_PM_BOLD
        let underline: CharacterUnderline = .SII_PM_UNDERLINE_CANCEL
        let reverse: CharacterReverse = .SII_PM_REVERSE_CANCEL
        let font: CharacterFont = .SII_PM_FONT_A
        let scale: CharacterScale = .SII_PM_VARTICAL_1_HORIZONTAL_1
        let alignment: PrintAlignment = .SII_PM_ALIGNMENT_CENTER
        var error: NSError?

        do {
            try? printerManager?.disconnect()

            try printerManager?.connect(
                Int32(SII_PM_PRINTER_MODEL_MP_B20),
                address: "MP-B20",
                portType: Int32(SII_PM_PRINTER_PORT_TYPE_BLUETOOTH)
            )

            var error: NSError?

            // 헤더 (볼드 + 가운데 정렬)
            let header = "納品書\n"
            
            _ = printerManager?.sendTextEx(
                header,
                bold: bold,
                underline: underline,
                reverse: reverse,
                font: font,
                scale: scale,
                alignment: alignment,
                error: &error //
            )

            // 본문
            let lines = text.components(separatedBy: "\n")
            for line in lines {
                let trimmed = line.trimmingCharacters(in: .whitespaces)
                if trimmed.isEmpty { continue }

                var formattedLine = trimmed
                if trimmed.contains(":") {
                    let parts = trimmed.components(separatedBy: ":")
                    if parts.count == 2 {
                        let name = parts[0].trimmingCharacters(in: .whitespaces)
                        let qty = parts[1].trimmingCharacters(in: .whitespaces)
                        let padded = name.padding(toLength: 24, withPad: " ", startingAt: 0)
                        formattedLine = "\(padded)\(qty)"
                    }
                }

                _ = printerManager?.sendTextEx(
                    header,
                    bold: bold,
                    underline: underline,
                    reverse: reverse,
                    font: font,
                    scale: scale,
                    alignment: alignment,
                    error: &error
                )

            }

            // 줄바꿈
            _ = printerManager?.sendText("\n\n", &error)

            try printerManager?.disconnect()

            print("✅ 印刷完了(sendTextEx)")
            notifyJavaScript(message: "✅ 印刷完了(sendTextEx)")
        } catch {
            print("❌ 印刷失敗: \(error)")
            notifyJavaScript(message: "❌ 印刷失敗。プリンターの電源や接続を確認してください。")
        }
    }

이 스위프트 코드에서 에러나오고잇어
- (BOOL)sendTextEx:(NSString *)text bold:(CharacterBold)bold underline:(CharacterUnderline)underline reverse:(CharacterReverse)reverse font:(CharacterFont)font scale:(CharacterScale)scale alignment:(PrintAlignment)alignment :(NSError **)error;
{
    @try {
        [_sdk sendTextEx:text bold:bold underline:underline reverse:reverse font:font scale:scale alignment:alignment];
        return true;
    } @catch (SIIPrinterException *e) {
        if (error) {
            *error = [self convertSIIPrinterExceptionToNSError:e];
        }
        return false;
    }
}
이게 .m코드고 
- (BOOL)sendTextEx:(NSString *_Nullable)text bold:(CharacterBold)bold underline:(CharacterUnderline)underline reverse:(CharacterReverse)reverse font:(CharacterFont)font scale:(CharacterScale)scale alignment:(PrintAlignment)alignment :(NSError *_Nullable*_Nullable)error;

이게 .h야

```
