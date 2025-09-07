# Async Callback Processing Flow

This diagram shows how async callbacks are processed in parallel for multiple PBE instances.

```mermaid
sequenceDiagram
    participant BS as km_bs_send_req
    participant CB as km_be_star_send_commit_id_cb
    participant KUIC as km_be_kuic_mgr_access_pbe_bs
    participant Dec as km_be_star_dec_num_target_inflights
    participant Mutex as Mutex/Condition Variable
    participant PBE1 as Remote PBE 1
    participant PBE2 as Remote PBE 2
    participant PBE3 as Remote PBE N

    Note over BS,PBE3: Async callback processing (parallel)
    
    par PBE 1 Response
        BS->>PBE1: Send commit ID message
        PBE1-->>BS: Response
        BS->>CB: Callback for PBE1 completion
        CB->>CB: Check command status
        alt Command failed
            CB->>CB: Log error
        else Command succeeded
            CB->>CB: Check aux info return status
            alt Aux info indicates error
                CB->>CB: Log PBE sync error
            end
        end
        CB->>KUIC: Release BS descriptor
        CB->>Dec: Decrement inflights counter
        Dec->>Mutex: Lock(&star->targets.mux)
        Dec->>Mutex: star->targets.inflight--
        alt inflight == 0
            Dec->>Mutex: Signal(&star->targets.cond)
        end
        Dec->>Mutex: Unlock(&star->targets.mux)
        
    and PBE 2 Response
        BS->>PBE2: Send commit ID message
        PBE2-->>BS: Response
        BS->>CB: Callback for PBE2 completion
        CB->>CB: Process response status
        CB->>KUIC: Release BS descriptor
        CB->>Dec: Decrement inflights counter
        Dec->>Mutex: Lock(&star->targets.mux)
        Dec->>Mutex: star->targets.inflight--
        alt inflight == 0
            Dec->>Mutex: Signal(&star->targets.cond)
        end
        Dec->>Mutex: Unlock(&star->targets.mux)
        
    and PBE N Response
        BS->>PBE3: Send commit ID message
        PBE3-->>BS: Response
        BS->>CB: Callback for PBE3 completion
        CB->>CB: Process response status
        CB->>KUIC: Release BS descriptor
        CB->>Dec: Decrement inflights counter
        Dec->>Mutex: Lock(&star->targets.mux)
        Dec->>Mutex: star->targets.inflight--
        alt inflight == 0
            Dec->>Mutex: Signal(&star->targets.cond)
        end
        Dec->>Mutex: Unlock(&star->targets.mux)
    end
    
    Note over Mutex: All operations complete when<br/>inflight counter reaches 0
```