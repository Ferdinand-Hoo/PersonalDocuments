flowchart TB

  

%% ===== Vehicle Device Layer =====

subgraph L1["车机设备层"]

direction TB

V1[Linux IVI System]

V2[Bluetooth Controller]

V3[HCI Log Capture]

end

  

%% ===== Log Collection Layer =====

subgraph L2["日志采集层"]

direction TB

C1[Tabby Terminal]

C2[Tabby MCP Server]

end

  

%% ===== HCI Parsing Layer =====

subgraph L3["日志解析层"]

direction TB

P1[HCI Packet Parser]

P2[Protocol Analysis Layer]

P3[Connection Timeline Builder]

end

  

%% ===== Skill Analysis Layer =====

subgraph L4["Skill分析层"]

direction TB

S1[Bluetooth Connection Skill]

S2[HiCar Analysis Skill]

S3[Carlink Analysis Skill]

S4[Knowledge Base]

end

  
  

L1 -->|SSH| L2

L2 --> L3

L3 --> L4