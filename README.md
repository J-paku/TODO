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
