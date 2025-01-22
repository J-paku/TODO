
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
# TODO EPPO Ver0.2




[screen-recording (1).webm](https://github.com/J-paku/TODO/assets/127849015/0e0d2eb1-deed-436b-802f-1c96034ba6e5)
