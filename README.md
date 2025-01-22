
```mermaid
graph TD
    subgraph Rack1["ラック1"]
        Hub1["ハブ1"]
        Hub2["ハブ2"]
    end

    subgraph Rack2["ラック2"]
        HubA["ハブA"]
        HubB["ハブB"]
    end

    Rack1 --|Port 50|--> Rack2
    Rack2 --|Port 50|--> Rack1
# 섹스
