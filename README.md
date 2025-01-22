
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
