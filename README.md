# DigitalLogicSimCommunityEdition
A collaborative platform to build digital systems from basic logic gates

# Digital Logic Sim
Digital logic sim is a playground that lets you start with a nand gate and build anything you want from a digital perspective, there have been a ton of really cool projects, from building RISC-V processors, GPU's etc within the simulator. Its an excellent tool to help you get the fundamentals of digital systems down, and really understand how things work.

## The problems

### Limited collaborative features

Currently there is really only one way to collaborate and thats by sharing files that DLS creates when you are working on a design.

### non-standard logic elements

DLS does not use the standard IEC logic elements, each element you create is just a box with some number of inputs and outputs. This is fine, and you can name the blocks, but It would be better to use standard IEC symbols for the atomic logic gates (AND, NOT, XOR etc.)

This would be tricky to implement if we just give the users nand gates only at the start of the sim, but I dont see why we would need to do that.

### Inability to rotate logic elements

DLS does not have the ability to rotate logic elements, currently everything is Left to right. It would be helpful to be able to rotate logic elements so that inputs and outputs could be on any side of the elements that are created.

### The UX is not great

The UX of DLS, while VERY useable could use a lot of love and polish. There are some really nice keyboard shortcuts but you cannot find what they are in the game itself (I dont think?) Also you cant customize keyboard shortcuts. It would be nice to have things like, when you create a new element/chip you could assign it to some hotkey, or maybe we have a grid of hotkeys that act as menus you click through etc? 

Another issue with the UX is that if you create a wire and then click in a place to create a segment of a wire, then cancel or hit escape the entire wire gets deleted instead of allowing the segment to just exist and be connected later.

## Technology choices

So first off, I would like this to be a web based project. I think this would be the easiest path forward for a few reasons:

1. Making it a web-based project allows the project to be platform agnostic, anyone with a browser can access/use the platform.
2. Due to being cross platform it will be much easier for users to collaborate with each other.
3. Allows the widest range of contributors, I believe that there are more devs in the web space than in other areas, so there will be a wider range of talent to help the project.

That being said, going forward, Im always open to hear why you might disagree with any of my takes, or decisions as I want this project to be the best it can be so always feel free to discuss this.

So with all that being said here is what im thinking:

# Architecture and Tech Stack

Our simulator decouples the visual layout (UI/Collaboration) from the execution layer (Simulation Engine). This ensures that heavy clock-cycle calculations and real-time multiplayer syncing do not bottleneck user interaction or cause UI lag.

```text
                  ┌─────────────────────────────────────────┐
                  │       Bun (Runtime & Build Tool)        │
                  └────────────────────┬────────────────────┘
                                       │
            ┌──────────────────────────┴─────────────────────────────────┐
            ▼                                                            ▼
┌───────────────────────┐                                     ┌───────────────────────┐
│     Client (UI)       │                                     │    Backend Server     │
├───────────────────────┤                                     ├───────────────────────┤
│ • React + TypeScript  │                                     │ • Bun.serve()         │
│ • TanStack Start (SSR)│ ◄── [Bi-directional WebSockets] ──► │ • Yjs WebSocket Server│
│ • React Flow (Canvas) │       (CRDT Layout Deltas)          │ • Rooms/State Manager │
│ • Yjs (CRDT Sync)     │                                     └───────────────────────┘
└───────────┬───────────┘
            │ (Local Compilation)
            ▼
┌───────────────────────┐
│  Simulation Engine    │
├───────────────────────┤
│ • Pure Vanilla TS     │
│ • Bitwise ArrayBuffers│
│ • Web Worker Thread   │
└───────────────────────┘
```



## Core Runtime & InfrastructureBun (Runtime, Package Manager, & Bundler)

### [Bun](https://tanstack.com/start/latest)
Replaces Node.js across the entire stack. Bun acts as our all-in-one development engine, serving as a lightning-fast package manager, a native TypeScript execution environment, and our production bundler.

### [Bun Native WebSockets](https://bun.com/docs/runtime/http/websockets)
Powers our multiplayer backend. Bun’s built-in WebSocket implementation is written in Zig/C++, allowing a single lightweight server instance to handle thousands of concurrent users syncing circuit data with minimal memory usage and sub-millisecond latency.

## Frontend and Routing

### React & TypeScript
Provides a component-based UI for managing workspace panels, chip selection menus, sidebar configurations, and strict data-typing for complex circuit schemas.

### [TanStack Start](https://tanstack.com/start/latest)
Our full-stack meta-framework. It handles file-based routing, server-side rendering (SSR) for fast initial workspace loads, and type-safe data loaders to fetch saved custom IC blueprints from database schemas.

## Visual Canvas

### (React Flow)[https://reactflow.dev/]
Manages the infinite graph canvas, interactive nodes (gates/ICs), draggable edges (wires), zooming, panning, and spatial layout calculations. It allows us to build an incredibly polished canvas UI out of the box.

## Real-Time Collaboration

### [Yjs](https://yjs.dev/) (CRDT Ecosystem)
A high-performance Conflict-free Replicated Data Type framework. It synchronization-locks the canvas nodes and edges arrays. If multiple users drag chips or delete wires simultaneously, Yjs automatically resolves conflicts deterministically without requiring a heavy centralized database authority.

### [y-websocket](https://github.com/yjs/y-websocket) (Communication Provider)
A lightweight binary communication protocol that streams Yjs document deltas back and forth over our native Bun WebSocket connections.

## Deterministic Simulation Engine

### Vanilla TypeScript (Decoupled Engine)
The actual logic gate simulation logic completely bypasses React, TanStack, and React Flow. It runs pure, non-allocating math logic on flat, compiled arrays.

### Web Workers API
Runs the simulation loop on a separate hardware thread. If a user builds a 4-bit computer with a clock running at high frequencies, the circuit evaluates inside a background Worker thread, ensuring the main browser thread remains at a locked 60 FPS for smooth dragging and panning.

### TypedArrays (Uint8Array / ArrayBuffer)
Wire states (High, Low, Floating) and gate logic connections are stored in raw binary memory buffers. This mimics hardware logic closely and avoids the garbage collection overhead associated with standard JavaScript objects during high-frequency execution.
