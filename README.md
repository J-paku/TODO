import re

data = [
    "N2018995 <==>|LAN ⇄ GE0| N2046376",
    "N2050402 <==>|Port 45 ⇄ Port 1| N2051329",
    "N2046376 <==>|GE0 ⇄ LAN| N2018995",
    "N2051329 <==>|Port 1 ⇄ Port 45| N2050402",
    "N2051610 <==>|Port 33 ⇄ LAN1| N2051564",
    "N2051333 <==>|Port 1 ⇄ Port 47| N2051294",
    "N2051294 <==>|Port 47 ⇄ Port 1| N2051333",
    "N2051564 <==>|LAN1 ⇄ Port 33| N2051610"
]

# ⇄ 기준 왼쪽에 있는 값을 추출
left_side_values = set()
for entry in data:
    match = re.match(r"(\S+) <==>\|(.+?) ⇄ (.+?)\| (\S+)", entry)
    if match:
        left_id, left_label, right_label, right_id = match.groups()
        left_side_values.add(left_id)
        left_side_values.add(left_label.strip())

# 삭제할 데이터 찾기
filtered_data = []
for entry in data:
    match = re.match(r"(\S+) <==>\|(.+?) ⇄ (.+?)\| (\S+)", entry)
    if match:
        left_id, left_label, right_label, right_id = match.groups()
        right_condition = f"{right_label.strip()}| {right_id}"
        if right_id in left_side_values and right_condition in entry:
            continue  # 삭제 조건에 맞으면 건너뛴다.
        filtered_data.append(entry)

# 결과 출력
for line in filtered_data:
    print(line)