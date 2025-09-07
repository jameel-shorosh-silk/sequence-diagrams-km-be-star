# Main Commit Flow Sequence Diagram

This diagram shows the high-level flow from commit completion to sending commit IDs to other PBE instances.

```mermaid
sequenceDiagram
    participant App as Application
    participant Star as km_be_star.c
    participant Topology as km_topology.c
    participant KUIC as km_be_kuic_mgr.c
    participant BS as km_bs.c
    participant Mutex as km_mutex.c
    participant PBE as Remote PBE Instances

    App->>Star: Commit completion request
    Note over Star: Process journal metadata<br/>Send AL sync if needed<br/>Notify post-commit callback
    
    Star->>Star: km_be_star_send_commit_id(star)
    
    Star->>Star: Check if in PBE instance
    alt Not in PBE instance
        Star-->>Star: return (early exit)
    end
    
    Note over Star,Mutex: Wait for previous operations to complete
    Star->>Star: km_be_star_wait_4_send_commit_id_completion(star)
    Star->>Mutex: Lock(&star->targets.mux)
    loop while star->targets.inflight > 0
        Star->>Mutex: Wait(&star->targets.cond, &star->targets.mux)
    end
    Star->>Mutex: Unlock(&star->targets.mux)
    
    Star->>Star: Increment initial inflights counter
    Star->>Mutex: Lock(&star->targets.mux)
    Star->>Mutex: star->targets.inflight++
    Star->>Mutex: Unlock(&star->targets.mux)
    
    Star->>Topology: Get topology and scan BE entries
    Topology-->>Star: For each BE entry
    
    loop For each PBE instance
        Star->>Star: Skip if not PBE type or same logical ID
        Star->>KUIC: Access PBE BS descriptor
        KUIC-->>Star: bs_desc
        
        alt bs_desc available
            Star->>Star: Set target context and aux info
            Star->>BS: Send async request
            BS->>PBE: Send commit ID message
            Star->>Star: Increment inflights counter
            Star->>Mutex: Lock(&star->targets.mux)
            Star->>Mutex: star->targets.inflight++
            Star->>Mutex: Unlock(&star->targets.mux)
        else bs_desc is NULL
            Star->>Star: Log error and continue
        end
    end
    
    Star->>Star: Decrement initial inflights counter
    Star->>Mutex: Lock(&star->targets.mux)
    Star->>Mutex: star->targets.inflight--
    Star->>Mutex: Signal condition if inflight == 0
    Star->>Mutex: Unlock(&star->targets.mux)
    
    Star-->>Star: Function returns (async callbacks continue)
    Star->>Star: Increment closed commits counter
    Star-->>App: Commit done
```