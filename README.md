# TODO EPPO Ver0.2
graph TD
    subgraph Rack1["ラック１"]
        Hub1["ハブ１"]
        Hub2["ハブ２"]
    end

    subgraph Rack2["ラック２"]
        HubA["ハブA"]
        HubB["ハブB"]
    end

    Rack1 -- "ポート：５０" --> Rack2
    Rack2 -- "ポート：５０" --> Rack1






[screen-recording (1).webm](https://github.com/J-paku/TODO/assets/127849015/0e0d2eb1-deed-436b-802f-1c96034ba6e5)
