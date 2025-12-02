### The Trusted Dealer Generation (TDG) Process

```mermaid
sequenceDiagram
    participant Dealer
    participant Members

    Dealer->>Dealer: Create Secret
    Dealer->>Dealer: Shard Secret
    Dealer-->>Members: Share Shares (Private)
    Dealer->>Dealer: Erase Secret & Shares
```

### The Distributed Key Generation (DKG) Process

```mermaid
sequenceDiagram
    participant Member
    participant Others

    Note left of Member: PREP
    Member<<->>Others: Determine m & n

    Note left of Member: ROUND 1
    Member->>Member: Create Secret
    Member->>Others: Share Commitment (Broadcast)

    Note left of Member: ROUND 2
    Others->>Others: Calc Signing Share
    Others-->>Member: Share Commitment (Private)
```

### Two-Round Signing

```mermaid
sequenceDiagram
    participant Coordinator
    participant Members

    Note left of Coordinator: ROUND 1
    Members->>Members: Create Nonces
    Members->>Members: Create Commitments
    Members-->>Coordinator: Share Commitments

    Note left of Coordinator: ROUND 2
    Coordinator->>Members: Share Message
    Coordinator->>Members: Share Commitments
    Members->>Members: Sign Message
    Members-->>Coordinator: Share Signature Share

    Note left of Coordinator: POST
    Coordinator->>Coordinator: Aggregate Signature
```

### Signing with Pre-Processing

```mermaid
sequenceDiagram
    participant Server
    participant Coordinator
    participant Members

    Note left of Server: PRE
    Members->>Members: Create Nonces
    Members->>Members: Create Commitments
    Members-->>Server: Share Commitments

    Note left of Server: ROUND 1
    Coordinator-->>Server: Retrieve Commitments
    Coordinator->>Members: Share Message
    Coordinator->>Members: Share Commitments
    Members->>Members: Sign Message
    Members-->>Coordinator: Share Signature Share

    Note left of Coordinator: POST
    Coordinator->>Coordinator: Aggregate Signature
```

    Note left of Member: POST
    Member->>Member: Finalize
```
