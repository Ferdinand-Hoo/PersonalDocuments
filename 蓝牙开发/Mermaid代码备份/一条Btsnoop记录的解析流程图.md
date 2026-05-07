flowchart TD

    A["一条 BTSnoop 记录record.payload"] --> B{包体首字节record.flags?}

    B -->|"1 (CMD)"| C["parse_hci_command→ HCI 命令"]

    B -->|"4 (EVENT)"| D["parse_hci_event→ HCI 事件"]

    B -->|"2 (ACL_DATA)"| E["parse_hci_acl→ ACL 头 + L2CAP 头"]

    B -->|"3 或其他"| R["raw_hex"]

    C --> Z["record.parsed"]

    D --> Z

    E --> F{cid?}

    F -->|"0x0001"| G["parse_l2cap_signaling→ L2CAP 信令"]

    F -->|"0x0004"| H["parse_att→ ATT"]

    F -->|"其他"| I["cid_hex + payload_hex"]

    G --> Z

    H --> Z

    I --> Z

    R --> Z