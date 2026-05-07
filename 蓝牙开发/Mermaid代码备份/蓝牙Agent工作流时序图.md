
sequenceDiagram

  
participant Cursor as Cursor Client

participant MCP as Tabby MCP Server

participant Tabby as Tabby Terminal

participant IVI as Linux IVI System

participant Parser as Log Parser

participant Skill as Analysis Skill

participant AI as AI Analyzer

  

Cursor->>MCP: Request HCI Log Capture

  

MCP->>Tabby: Execute Terminal Command

Tabby->>IVI: SSH Connect

  

IVI-->>Tabby: Connection Established

  

Tabby->>IVI: Capture HCI log

IVI-->>Tabby: Return HCI Log File

  

Tabby-->>MCP: Provide Log Data

MCP-->>Cursor: Return HCI Log

  

Cursor->>Parser: Load HCI Log

Parser->>Parser: Parse HCI Packets

Parser->>Parser: Decode Command/Event/ACL

Parser->>Parser: Protocol Analysis

  

Parser->>Skill: Build Connection Timeline

  

Skill->>AI: Invoke Analysis Skill

AI->>AI: Reasoning with Knowledge Base

  

AI-->>Cursor: Generate Diagnosis Report