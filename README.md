PrintMaster에서 BLE 신호를 만들어내는 과정은 다음과 같은 순서일 가능성이 높다:

BLE 디바이스 스캔 (Discovery):

PrintMaster 앱은 iOS의 CoreBluetooth 프레임워크를 사용하여 BLE 디바이스를 스캔한다.

CBPeripheral 객체를 통해 주변 장치를 탐색하고 식별할 수 있다.

이 과정에서 장치의 UUID, RSSI, 서비스 UUID 등이 캡처된다.

BLE 연결 (Connection):

앱이 대상 BLE 장치를 식별한 후, connect(_:options:) 메서드로 연결을 시도한다.

이 과정에서 장치의 GATT 서비스 및 특성 (Characteristics)을 검색하게 된다.

서비스 및 특성 탐색 (Service and Characteristic Discovery):

연결이 완료되면, 장치의 GATT 서버에서 제공하는 서비스 및 특성을 탐색한다.

이 과정에서 특정 Characteristic의 UUID를 통해 데이터를 송수신할 핸들(예: 0x0008)을 식별할 수 있다.

데이터 전송 (Data Transmission):

PrintMaster는 writeValue(_:for:type:) 메서드를 사용하여 데이터를 withoutResponse 타입으로 전송하고 있다.

withoutResponse는 BLE의 Write Without Response 옵션으로, 성공 여부에 대한 피드백을 받지 않는다.

이 과정에서 실제로 전송되는 데이터는 패킷 단위로 나누어 전송된다.

BLE 데이터 포맷 (Data Format):

인쇄 명령은 ESC/POS 또는 기타 프린터 전용 프로토콜 형식으로 인코딩된다.

예를 들어, 특정 텍스트를 인쇄하려면 0x1B 0x40 (초기화 명령)과 같은 ESC/POS 명령이 전송될 수 있다.

이 명령은 BLE 패킷의 0x0008 핸들에 기록된다.

로그 메시지 분석:

로그에서 Writing value without response to characteristic handle 0x0008이라는 메시지가 반복적으로 출력되었다.

이 메시지는 PrintMaster가 0x0008 핸들에 데이터를 전송하고 있음을 의미한다.

이 핸들은 PrintMaster와 프린터 간의 실제 데이터 송수신 채널로 보인다.

✅ 핸들 0x0008의 역할:
이 핸들은 GATT 서비스 내의 특정 Characteristic에 매핑된 것으로 보인다.

PrintMaster는 이 핸들에 인쇄 명령을 전송함으로써 BLE 프린터에 인쇄 데이터를 전달하고 있다.

✅ 데이터 전송 패킷의 구조:
0x0008에 전송되는 데이터는 일반적으로 다음과 같은 형식일 가능성이 높다:

[명령 프리픽스] + [데이터 길이] + [데이터 내용] + [종료 바이트]

이제 이 데이터를 캡처하고 해석하면 PrintMaster가 어떤 명령을 전송하는지 알 수 있다.
내일 이 부분을 좀 더 집중적으로 분석하면 될 것 같다.
