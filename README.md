
```mermaid
graph LR
    subgraph Rack1["ラック１\n\n\n\n\n\n"]
        direction LR
        Hub1["U35：裏：XXX"]
    end

    subgraph Rack2["ラック２\n\n\n\n\n"]
        direction LR
        HubA["U32：裏：XXX"]
        HubB["UXX：裏：XXX"]
    end

    Hub1 -- "Port：50 ⇄ Port32" --> HubA
    Hub1 -- "Port：13 ⇄ Port32" --> HubB
# 섹스

import pandas as pd

# Mermaid 코드 데이터 구조 정의
data = {
    "Rack": ["Rack1", "Rack1", "Rack2", "Rack2", "Rack2"],
    "U_Position": ["U35", "U35", "U32", "UXX", "U32"],
    "Side": ["裏", "裏", "裏", "裏", "裏"],
    "Connection": ["Port:50 → Port32", "Port:13 → Port32", "", "", ""],
    "Target": ["Rack2:U32", "Rack2:UXX", "", "", ""]
}

# 데이터 프레임 생성
df = pd.DataFrame(data)

# Mermaid 코드 생성
mermaid_code = """graph LR
    subgraph Rack1["ラック１\\n\\n\\n\\n\\n"]
        direction LR
        Hub1["U35：裏：XXX"]
    end

    subgraph Rack2["ラック２\\n\\n\\n\\n"]
        direction LR
        HubA["U32：裏：XXX"]
        HubB["UXX：裏：XXX"]
    end

    Hub1 -- "Port：50 → Port32" --> HubA 
    Hub1 -- "Port：13 → Port32" --> HubB
"""

# 파일로 저장
file_path = "PakuTestText.txt"
with open(file_path, "w", encoding="utf-8") as file:
    file.write(mermaid_code)

print(f"Mermaid 코드가 {file_path} 파일로 저장되었습니다.")
