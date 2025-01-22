
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

    subgraph Rack3["ラック３\n\n\n\n\n"]
        direction LR
        HubC["UYY：裏：XXX"]
    end

    Hub1 -- "Port：50 ⇄ Port32" --> HubA
    Hub1 -- "Port：13 ⇄ Port32" --> HubB
    HubB -- "Port：25 ⇄ Port10" --> HubC

# 섹스

import pandas as pd

# データベースから取得したデータ（サンプル）
data = [
    {"Rack": "Rack1", "U_Position": "U35", "Side": "裏", "Hub": "Hub1", "Port": "Port：50 ⇄ Port32", "TargetRack": "Rack2", "TargetHub": "HubA"},
    {"Rack": "Rack1", "U_Position": "U35", "Side": "裏", "Hub": "Hub1", "Port": "Port：13 ⇄ Port32", "TargetRack": "Rack2", "TargetHub": "HubB"},
    {"Rack": "Rack2", "U_Position": "U32", "Side": "裏", "Hub": "HubA"},
    {"Rack": "Rack2", "U_Position": "UXX", "Side": "裏", "Hub": "HubB"},
    {"Rack": "Rack3", "U_Position": "UYY", "Side": "裏", "Hub": "HubC"},  # Rack3 추가
    {"Rack": "Rack2", "U_Position": "UXX", "Side": "裏", "Hub": "HubB", "Port": "Port：25 ⇄ Port10", "TargetRack": "Rack3", "TargetHub": "HubC"},  # Rack2와 Rack3 연결
]

# DataFrameに変換
df = pd.DataFrame(data)

# Mermaidコードの作成
mermaid_code = "graph LR\n"

# ラックごとにグループ化
racks = df.groupby("Rack")

for rack_name, group in racks:
    mermaid_code += f'    subgraph {rack_name}["{rack_name}\\n\\n\\n\\n"]\n'
    mermaid_code += "        direction LR\n"

    # ハブを追加
    for _, row in group.iterrows():
        hub_id = row["Hub"]
        position = row["U_Position"]
        side = row["Side"]
        mermaid_code += f'        {hub_id}["{position}：{side}：XXX"]\n'

    mermaid_code += "    end\n\n"

# ハブ間の接続을 추가
for _, row in df.iterrows():
    if "Port" in row and pd.notna(row["Port"]) and "TargetRack" in row and pd.notna(row["TargetRack"]):
        source_hub = row["Hub"]
        target_hub = row["TargetHub"]  # 원래 Hub 명칭 유지
        port_info = row["Port"]
        mermaid_code += f'    {source_hub} -- "{port_info}" --> {target_hub}\n'

# Mermaidコードの出力
print(mermaid_code)


