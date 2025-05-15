1. Xcode에서 새로운 Swift 프로젝트 생성하기
Xcode 열기 → Create a new project → App

프로젝트 이름: MyWebApp

언어: Swift

Interface: Storyboard 또는 SwiftUI (SwiftUI 추천)


2. Info.plist 설정하기
Info.plist 파일에서 App Transport Security 설정을 추가해야 함.

Info.plist에 추가:
```
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

3. ViewController.swift 코드 작성
WebView를 추가하고 PWA URL을 로드하도록 설정.

ViewController.swift:
```
import UIKit
import WebKit

class ViewController: UIViewController, WKNavigationDelegate {
    
    var webView: WKWebView!

    override func viewDidLoad() {
        super.viewDidLoad()

        // WebView 설정
        let webConfiguration = WKWebViewConfiguration()
        webConfiguration.allowsInlineMediaPlayback = true
        webView = WKWebView(frame: self.view.frame, configuration: webConfiguration)
        webView.navigationDelegate = self
        view.addSubview(webView)

        // PWA URL
        let pwaURLString = "https://your-pwa-url.com"
        if let url = URL(string: pwaURLString) {
            let request = URLRequest(url: url)
            webView.load(request)
        }
    }
}
```

4. AppDelegate.swift에 WebView 관련 설정 추가
만약 Background 작업이나 Bluetooth 통신을 나중에 추가할 경우,
AppDelegate에서 초기화하는 코드도 추가할 수 있다.

```
import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {

        window = UIWindow(frame: UIScreen.main.bounds)
        window?.rootViewController = ViewController()
        window?.makeKeyAndVisible()

        return true
    }
}
```
5. Bluetooth 기능을 나중에 추가하기 위한 구조 설계
현재는 WebView만 로드하는 구조이지만,
나중에 Bluetooth 연결 로직을 추가할 때는 BluetoothManager.swift 파일을 생성하여 확장할 수 있음.

BluetoothManager.swift (기본 구조):
```
import Foundation
import CoreBluetooth

class BluetoothManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {

    var centralManager: CBCentralManager?

    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
        case .poweredOn:
            print("Bluetooth is ON")
            // 나중에 연결 로직 추가
        default:
            print("Bluetooth is not available")
        }
    }
}
```

6. iOS에서 WebView로 데이터 전송 (JavaScript ↔ Native 통신)
PWA에서 iOS로 명령어를 전달하거나 Bluetooth 연결 명령을 보낼 수 있는 구조를 만들어두면 나중에 확장하기 쉽다.

ViewController.swift에 추가:
```
extension ViewController: WKScriptMessageHandler {
    
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if message.name == "nativeCommand", let command = message.body as? String {
            print("Received command from PWA: \(command)")
            // Bluetooth 관련 명령어 처리
        }
    }
}
```

PWA에서 JavaScript로 명령어 전송:
```
window.webkit.messageHandlers.nativeCommand.postMessage("connectBluetooth");
```

 정리 및 최종 구조:
PWA URL은 Swift WebView에서 로드.

Bluetooth 통신 로직은 나중에 BluetoothManager.swift에서 구현.

JavaScript ↔ Swift 통신 구조를 미리 만들어둠으로써, PWA에서 Bluetooth 명령을 보낼 준비를 해둠.

다음 단계:
PWA URL이 HTTPS인지 확인 (iOS에서는 HTTPS가 필수임).

Bluetooth 통신을 위한 데이터 패킷 구조 설계.

Phomemo M120에서 전송하는 패킷 구조를 파악한 후, 해당 패킷을 전송할 BluetoothManager.swift 로직 구현.

이제 이 구조를 기본으로 잡고, Bluetooth 통신 로직을 추가할 준비를 하면 돼.
어느 부분부터 시작할지 말해주면 거기부터 바로 시작하자. ✅🙂👍
