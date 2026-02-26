# Accord — Architecture Diagrams

---

## 1. System Architecture Overview

```mermaid
graph TD
    Client["Client (Flutter)"]
    Library["Library (Rust)"]
    FullNode["Full Node (Rust / libp2p)"]
    GossipNode["Gossip Node (Rust / libp2p)"]
    Network["P2P Network"]

    Client -->|"calls"| Library
    Library -->|"calls"| FullNode
    FullNode -->|"participates in"| Network
    GossipNode -->|"participates in"| Network
    Network -->|"peer sync"| FullNode
    Network -->|"peer sync"| GossipNode
```

---

## 2. Peer Discovery

Both Full and Gossip nodes use the same two-pronged discovery strategy.

```mermaid
flowchart TD
    Start(["Node starts"])
    Start --> Local & Remote

    Local["mDNS scan (local network)"]
    Remote["Kademlia DHT scan (remote / internet)"]

    Local --> Ports["Port range scan 51030 – 51042"]
    Remote --> Ports

    Ports --> Found{"Peers found?"}
    Found -->|yes| Store["Add to peers list ($HOME/.accord/peers)"]
    Found -->|no| Retry["Retry / keep listening"]
    Store --> Connect["Establish libp2p connections"]
```

---

## 3. Full Node — Storage Layout

```mermaid
graph TD
    Root["$HOME/.accord"]

    Root --> Peers["peers (vector of peer addresses)"]
    Root --> Database["database/ (merkle tree storage)"]
    Root --> Current["current (active root hash pointer)"]
    Root --> Indexes["indexes (merkle tree of root hashes)"]
    Root --> Users["users/ (all known users)"]
    Root --> User["user/ (local user identity)"]

    Database --> HashDir["&lt;root hash&gt;/"]
    HashDir --> Data["data (full merkle tree)"]

    Users --> UserId["&lt;user id&gt;/"]
    UserId --> UserMeta["meta (JSON metadata)"]

    User --> Connections["connections/"]
    User --> LocalId["id (elliptic curve user ID)"]
    User --> LocalMeta["meta (JSON metadata)"]
    User --> MasterPriv["private (master private key)"]
    User --> MasterPub["public (master public key)"]

    Connections --> ConnDir["&lt;connection user id&gt;/"]
    ConnDir --> ConnMeta["meta (JSON metadata)"]
    ConnDir --> ConnPriv["private (derived private key)"]
    ConnDir --> ConnPub["public (derived public key)"]
    ConnDir --> Shared["shared (DH shared key)"]
```

---

## 4. Cryptographic Key Hierarchy

```mermaid
graph TD
    EC["Elliptic Curve"]

    EC -->|"generates"| MasterPriv["Master Private Key (stored in user/private)"]
    MasterPriv -->|"derives"| MasterPub["Master Public Key (stored in user/public)"]
    MasterPub -->|"hashes to"| UserId["User ID (stored in user/id)"]

    MasterPriv -->|"derives per-connection"| ConnPriv["Connection Private Key (stored in connections/&lt;id&gt;/private)"]
    ConnPriv -->|"derives"| ConnPub["Connection Public Key (stored in connections/&lt;id&gt;/public)"]

    ConnPriv -->|"Diffie-Hellman exchange with remote public key"| Shared["Shared Secret Key (stored in connections/&lt;id&gt;/shared)"]
    RemotePub["Remote User's Connection Public Key"] -->|"input to DH"| Shared
```

---

## 5. Sync Workflows

### 5a. Outbound Sync — Node Initiates

```mermaid
sequenceDiagram
    participant Lib as Library
    participant Node as Full Node
    participant Store as Storage
    participant Net as Network / Peers

    Lib->>Node: sync data

    Node->>Store: read current root hash
    Node->>Net: send current root hash

    alt Network already has latest
        Net-->>Node: Latest (no update needed)
        Node-->>Lib: Done
    else Network has newer data
        Net-->>Node: Update — send merkle tree
        Node->>Store: get root hash of received tree
        Node->>Store: create directory for new root hash
        Node->>Store: store tree in directory
        Node->>Store: update current → new hash
        Node->>Store: update indexes with new hash
        Node-->>Lib: Done
    else Node has newer data
        Node->>Net: send update with current tree
        Node-->>Lib: Done
    end
```

### 5b. Inbound Sync — Peer Initiates

```mermaid
sequenceDiagram
    participant Net as Network / Peer
    participant Node as Full Node
    participant Store as Storage

    Net->>Node: sync request with hash

    Node->>Store: read current root hash
    Node->>Node: compare received hash vs current

    alt Hash matches current (already latest)
        Node->>Net: send back "latest" confirmation
    else Hash does not match current
        Node->>Store: read indexes
        alt Hash exists in indexes
            Node->>Net: send update with tree for that hash
        else Hash not found in indexes
            Node->>Net: send our current hash
            Net-->>Node: Update — send merkle tree
            Node->>Store: get root hash of received tree
            Node->>Store: create directory for new root hash
            Node->>Store: store tree in directory
            Node->>Store: update current → new hash
            Node->>Store: update indexes with new hash
        end
    end
```

---

## 6. Message Workflow

```mermaid
sequenceDiagram
    participant Lib as Library
    participant Node as Full Node
    participant Store as Storage
    participant Sync as Sync (Network)

    Lib->>Node: create message (message model)
    Node->>Store: store message in merkle tree
    Node->>Sync: sync data to peers
```

---

## 7. User Workflows

### 7a. Create User

```mermaid
sequenceDiagram
    participant Lib as Library
    participant Node as Full Node
    participant Store as Storage
    participant Sync as Sync (Network)

    Lib->>Node: create user

    Node->>Node: generate ID (elliptic curve)
    Node->>Store: store ID

    Node->>Node: create master private / public key pair
    Node->>Store: store master private / public key pair

    Node->>Sync: sync user ID with peers

    Node-->>Lib: return user model
```

### 7b. Get All Users

```mermaid
sequenceDiagram
    participant Lib as Library
    participant Node as Full Node
    participant Store as Storage

    Lib->>Node: get users
    Node->>Store: read all users (IDs + meta)
    Node->>Node: build user models from IDs and metadata
    Node-->>Lib: return list of user models
```

### 7c. Get Single User

```mermaid
sequenceDiagram
    participant Lib as Library
    participant Node as Full Node
    participant Store as Storage

    Lib->>Node: get user (id)
    Node->>Store: read user (ID + meta)
    Node->>Node: build user model from ID and metadata
    Node-->>Lib: return user model
```

---

## 8. Connection Workflow (DH Key Exchange)

```mermaid
sequenceDiagram
    participant Lib as Library
    participant NodeA as Full Node (initiator)
    participant StoreA as Storage (initiator)
    participant Sync as Sync (Network)
    participant NodeB as Full Node (remote user)
    participant StoreB as Storage (remote)

    Lib->>NodeA: create connection (user id)

    NodeA->>StoreA: create connection directory with user ID
    NodeA->>StoreA: generate connection private / public key pair
    NodeA->>Sync: share connection public key via message

    Sync->>NodeB: sync public key message

    NodeB->>NodeB: accept connection
    NodeB->>StoreB: create connection private / public key pair
    NodeB->>StoreB: create shared key (from NodeA's public key)
    NodeB->>Sync: send own connection public key via message

    Sync->>NodeA: sync NodeB's public key message

    NodeA->>StoreA: derive shared key from NodeB's public key
    Note over NodeA,NodeB: Both sides now hold matching shared secret (DH)
```
