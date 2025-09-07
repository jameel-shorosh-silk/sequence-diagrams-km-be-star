# Complete End-to-End Flow with Error Handling

This diagram shows the complete flow from application to commit completion, including all error handling paths.

```mermaid
sequenceDiagram
    participant App as Application
    participant Star as km_be_star.c
    participant Topology as km_topology.c
    participant KUIC as km_be_kuic_mgr.c
    participant BS as km_bs.c
    participant Mutex as km_mutex.c
    participant PBE as Remote PBE

    App->>Star: Commit completion request
    Star->>Star: km_be_star_send_commit_id(star)
    
    alt Not in PBE instance
        Star-->>Star: return (early exit)
    else In PBE instance
        Note over Star,Mutex: Wait for previous operations completion
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
        loop For each BE entry
            alt Not PBE type
                Star->>Star: Skip entry
            else Is PBE type
                alt Same logical ID
                    Star->>Star: Skip entry
                else Different logical ID
                    Star->>KUIC: Access PBE BS descriptor
                    alt bs_desc is NULL
                        KUIC-->>Star: NULL
                        Star->>Star: Log error and continue
                    else bs_desc available
                        KUIC-->>Star: bs_desc
                        Star->>Star: Set target context and aux info<br/>(commit_id, is_ddb, ini_logical_id)
                        Star->>BS: Send async request
                        alt Send failed
                            BS-->>Star: Error status
                            Star->>KUIC: Release BS descriptor
                            Star->>Star: Log error
                        else Send successful
                            BS-->>Star: Success/In Progress
                            Star->>Star: Increment inflights counter
                            Star->>Mutex: Lock(&star->targets.mux)
                            Star->>Mutex: star->targets.inflight++
                            Star->>Mutex: Unlock(&star->targets.mux)
                            
                            Note over BS,PBE: Async processing
                            BS->>PBE: Send commit ID message
                            PBE-->>BS: Response
                            BS->>Star: Completion callback (km_be_star_send_commit_id_cb)
                            Star->>Star: Process response and cleanup
                            Star->>KUIC: Release BS descriptor
                            Star->>Star: Decrement inflights counter
                            Star->>Mutex: Lock(&star->targets.mux)
                            Star->>Mutex: star->targets.inflight--
                            Star->>Mutex: Signal condition if inflight == 0
                            Star->>Mutex: Unlock(&star->targets.mux)
                        end
                    end
                end
            end
        end
        
        Star->>Star: Decrement initial inflights counter
        Star->>Mutex: Lock(&star->targets.mux)
        Star->>Mutex: star->targets.inflight--
        Star->>Mutex: Signal condition if inflight == 0
        Star->>Mutex: Unlock(&star->targets.mux)
        Star-->>Star: Function returns
    end
    
    Star->>Star: Increment closed commits counter
    Star-->>App: Commit done
```