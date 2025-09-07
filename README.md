# Sequence Diagrams for km_be_star_send_commit_id Function

This repository contains sequence diagrams that illustrate the flow of the `km_be_star_send_commit_id` function, which is responsible for sending commit IDs to remote PBE (Persistent Block Engine) instances.

## Overview

The `km_be_star_send_commit_id` function is a complex asynchronous system that:
- Synchronizes with previous operations using mutex/condition variables
- Scans topology to find remote PBE instances
- Sends commit IDs asynchronously to multiple PBE instances
- Handles callbacks and resource cleanup
- Manages inflight operation counters for proper synchronization

## Diagrams

### 1. [Main Commit Flow](01-main-commit-flow.md)
Shows the high-level flow from commit completion to sending commit IDs to other PBE instances.

### 2. [Synchronization Flow](02-synchronization-flow.md)
Details the synchronization mechanisms and counter management within the function.

### 3. [Async Callback Flow](03-async-callback-flow.md)
Illustrates how async callbacks are processed in parallel for multiple PBE instances.

### 4. [Complete Flow with Error Handling](04-complete-flow-with-error-handling.md)
Shows the complete end-to-end flow including all error handling paths.

## How to View the Diagrams

### Option 1: GitHub (Recommended)
Simply click on any of the markdown files above. GitHub automatically renders Mermaid diagrams.

### Option 2: Local Viewing
1. Clone this repository
2. Use any markdown viewer that supports Mermaid (VS Code with Mermaid extension, etc.)
3. Or use online tools like:
   - [Mermaid Live Editor](https://mermaid.live/)
   - [GitHub's built-in rendering](https://github.com/jameel-shorosh-silk/sequence-diagrams-km-be-star)

### Option 3: Export as Images
You can export these diagrams as PNG/SVG using:
- Mermaid CLI: `mmdc -i diagram.md -o diagram.png`
- Online tools like Mermaid Live Editor
- VS Code extensions

## Key Components

### Data Structures
- `struct km_be_star *star` - Main star object
- `struct km_be_star_target_ctxt` - Context for each target PBE
- `struct km_io_aux_info` - Auxiliary information sent with commit ID
- `km_bs_req` - Block storage request structure

### Synchronization Primitives
- `star->targets.mux` - Mutex protecting the inflight counter
- `star->targets.cond` - Condition variable for waiting on completion
- `star->targets.inflight` - Counter tracking outstanding async operations

### Key Functions
- `km_be_star_send_commit_id()` - Main function sending commit IDs
- `km_be_star_send_commit_id_cb()` - Async callback for request completion
- `km_be_star_wait_4_send_commit_id_completion()` - Wait for previous operations
- `km_be_star_inc_num_target_inflights()` - Increment inflight counter
- `km_be_star_dec_num_target_inflights()` - Decrement inflight counter and signal

## Flow Summary

1. **Wait Phase**: Wait for any previous send operations to complete
2. **Preparation Phase**: Increment inflight counter and scan topology
3. **Sending Phase**: Send async requests to all PBE instances
4. **Callback Phase**: Handle async completions and cleanup resources
5. **Synchronization**: Use mutex/condition variable to coordinate async operations

## Architecture Highlights

- **Asynchronous Design**: Main function returns quickly while callbacks continue processing
- **Thread Safety**: Proper mutex/condition variable usage for multi-threaded environment
- **Resource Management**: Careful acquisition and release of BS descriptors
- **Error Handling**: Comprehensive error checking and logging at each step
- **Scalability**: Handles multiple PBE instances in parallel

## Contributing

Feel free to suggest improvements to the diagrams or add additional views that might be helpful for understanding this complex system.