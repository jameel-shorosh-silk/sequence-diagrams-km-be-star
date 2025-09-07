# Synchronization and Counter Management

This diagram shows the detailed synchronization mechanisms and counter management within the `km_be_star_send_commit_id` function.

```mermaid
sequenceDiagram
    participant SendCommit as km_be_star_send_commit_id
    participant Wait as km_be_star_wait_4_send_commit_id_completion
    participant Inc as km_be_star_inc_num_target_inflights
    participant Dec as km_be_star_dec_num_target_inflights
    participant Mutex as Mutex/Condition Variable

    SendCommit->>Wait: Wait for previous operations
    Wait->>Mutex: Lock(&star->targets.mux)
    loop while star->targets.inflight > 0
        Wait->>Mutex: Wait(&star->targets.cond, &star->targets.mux)
    end
    Wait->>Mutex: Unlock(&star->targets.mux)
    Wait-->>SendCommit: Previous operations complete
    
    SendCommit->>Inc: Increment initial inflights counter
    Inc->>Mutex: Lock(&star->targets.mux)
    Inc->>Mutex: star->targets.inflight++
    Inc->>Mutex: Unlock(&star->targets.mux)
    Inc-->>SendCommit: Counter incremented
    
    Note over SendCommit: Process each PBE target
    
    loop For each successful request
        SendCommit->>Inc: Increment inflights counter
        Inc->>Mutex: Lock(&star->targets.mux)
        Inc->>Mutex: star->targets.inflight++
        Inc->>Mutex: Unlock(&star->targets.mux)
        Inc-->>SendCommit: Counter incremented
    end
    
    SendCommit->>Dec: Decrement initial inflights counter
    Dec->>Mutex: Lock(&star->targets.mux)
    Dec->>Mutex: star->targets.inflight--
    alt inflight == 0
        Dec->>Mutex: Signal(&star->targets.cond)
    end
    Dec->>Mutex: Unlock(&star->targets.mux)
    Dec-->>SendCommit: Initial counter decremented
```