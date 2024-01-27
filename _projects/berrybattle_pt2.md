---
layout: page
title: Berry Battle Pt.2
date: 2024-01-26 11:59:00-0000
description: gRPC Prototype and Performance Benchmarks
img: assets/img/berry-battle-logo-white-text.png
importance: 2
category: Berry Battle
giscus_comments: false
giscus_repo:
toc:
  sidebar: left
---
## Topics Covered
- Using gRPC with Rust
- Cargo Build Scripts
- Tests
- Performance benchmarking

## TLDR
[Game Simulator Source...](https://github.com/berrybattle/berry-battle-server/tree/v0.0.2)

[AI Node Source...](https://github.com/berrybattle/berry-battler/tree/v0.0.2)

## Background
Following the conception of the project the first problem I need to solve is communication between the server and AI nodes. I don't think something with direct access is viable since few people would open their hardware up to the world for direct ssh connections. So I need a higher level web API that is proven to be safe, language agnostic, performant, and preferably modern. I looked at REST, and HTTP with JSON before settling on gRPC which appears to meet my criteria.

The next step is figuring out how it can be implemented with Rust in a way that will work with my high level goals. Once I get basic communications running I will run some simple performance benchmarks to confirm the approach is feasible.

## Let's Tackle the Problem
### Defining the Protocol Buffer
I found several helpful examples for using gRPC with Rust and the general consensus seemed to be that the tonic crate was the way to go. The [tonic repository has quite a few working examples](https://github.com/hyperium/tonic/tree/master/examples) that really helped me get started. I won't go into tons of detail about gRPC specifically but will cover my journey from the tonic examples to my working prototype.

The first thing I wanted to do was establish some general data structures. To help protect against hacking or cheating I want the bytes coming in and out to be well defined and certainly not a binary blob. Tonic/gRPC handles this with protocol buffers that are defined in files with the ```.proto``` extension. My definitions can be spread logically across multiple files and then they are combined and compiled together in a pre-build process or what cargo calls a build script. Note that the build script doesn't actually do the compiling, instead it initiates a build with ```protoc``` which then compiles the protocol buffer into a intermediary ```*.rs``` file.

The project setup looks something like this:
```console
package-root
  ├── proto
  │     ├── data.rs
  │     └── services.rs
  ├── src
  │     ├── lib.rs
  │     └── main.rs
  ├── Cargo.toml
  └── build.rs

```

The build script had to be modified from the example so that it builds all my ```.proto``` files. In it we can see that the build script defines the output directory to place the code-generated ```*.rs``` files before compiling the protocol definition files with ```tonic_build```. I also prefix the ```file_descriptor_set_path``` with "bb_rpc_" which is the name of protocol definition package that I define shortly. 
```rust
// build.rs
use std::{env, path::PathBuf};

fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    tonic_build::configure()
        .build_client(true)
        .file_descriptor_set_path(out_dir.join("bb_rpc_descriptor.bin"))
        .compile(
            &["proto/data.proto", "proto/services.proto"],
            &["proto"],
        )
        .unwrap();
}
```

Now I was able to start experimenting with my data structures. I split the protocol buffer definition into two files based on their purpose. One file would contain the data model and the other would contain the communication services. 

For this prototype and benchmarking experiment I need the data to be representative of a simple but typical game object. It needed to define things like position, direction, enumerations for state, strings, and representations of time. It also needed to represent lists of data like a vector or an array. I ended up using the following as a starting point...
```console
// data.proto
syntax = "proto3";
package bb_grpc;
enum UpdateStatus {
      PROCESSING = 0;
      FINISHED = 1;
}
enum UnitType {
      UNKNOWN = 0;
      MOBILE = 1;
      STATIC = 2;
}
message UnitDirectionVector {
      float x = 1;
      float y = 2;
}
message UnitPosition {
      float x = 1;
      float y = 2;
      uint32 layer = 3;
      UnitDirectionVector direction = 4;
}
message UnitState {
      uint32 id = 1;
      UnitType unit_type = 2;
      UnitPosition position = 3;
      string tag = 4;
}
message UpdateRpcRequest {
      UpdateStatus status = 1;
      uint32 update_id = 2;
      repeated UnitState units = 3;
      uint64 per_unit_proc_time_ns = 4;
}
message UpdateRpcResponse {
      UpdateStatus updated_status = 1;
      uint32 update_id = 2;
      repeated UnitState units = 3;
      uint64 per_unit_proc_time_ns = 4;
      uint64 single_pass_elapsed_time_us = 5;
}
```

The protocol buffer definition files are compiled into rust files with syntax you would expect. The ```package``` field acts as a crate name when referenced in the ```*.rs``` files. The ```message``` prefix will compile the item into a ```struct``` object which I use to interact with in the application. Lists of objects are created with the ```repeated``` prefix as shown in both ```UpdateRpcRequest``` and ```UpdateRpcResponse```. Nesting is also intuitive as can be seen with the ```UnitDirectionVector``` nested within the ```UnitPosition``` which itself is nested in a ```UnitState``` object.

The servers and services are also easy to define. In this case I am using a single ```UpdateService ``` service which tells the gRPC server to listen for a ```UpdateRpc ``` event and reply with a ```UpdateRpcResponse``` which we will define in a handler.
```console
// services.proto
syntax = "proto3";
package bb_grpc;
import "data.proto";
service UpdateService {
  rpc UpdateRpc (UpdateRpcRequest) returns (UpdateRpcResponse);
}
```

### Handling the Update Event (Simulator - gRPC Client)
Both the simulator (gRPC client) and the AI node (gRPC server) share the same buffer protocol definition, build scripts, and general setup mentioned above. They differ in how they use the generated code since the simulator is requesting a service and the AI node is actually running the gRPC server.

The gRPC client code is relatively simple. I create a ```async``` function to perform the RPC and provide it with a ```UpdateRpcRequest``` as we defined in our ```.proto``` file. The client then tries to connect with the gRPC server and waits for a connection to be established. Once successful, the ```UpdateRpc``` event is triggered with our supplied data, ```current_state``` of type ```UpdateRpcRequest```. When the ```UpdateRpc``` call is complete, either a ```UpdateRpcResponse``` object or an Error is returned.

```rust
// berry-battle-server/src/main.rs
async fn request_updates(
    current_state: UpdateRpcRequest,
) -> Result<UpdateRpcResponse, Box<dyn std::error::Error>> {
    let mut client = UpdateServiceClient::connect("http://[::1]:50051").await?;

    match client.update_rpc(current_state).await {
        Ok(response) => Ok(response.into_inner()),
        Err(e) => Err(Box::new(e)),
    }
}
```

Packaging the data into the ```UpdateRpcRequest``` object just like instantiating any other struct in Rust and I will cover it a little later. That's it for the client gRPC side.

### Handling the Update Event (AI Node - gRPC Server)
The gRPC server takes a little more to setup but not much. We need to define a ```struct``` so that we can access our custom trait implementations like they were virtual functions. In this case I named it ```UpdateServiceTraitWrapper```. The ```UpdateService``` defined with the ```service``` keyword in the ```.proto``` file determines the name of the virtual function, or trait, that I need to make an implementation for. This trait implementing struct can be named whatever you want as long as the implementation defined below it matches the service name in the ```*.proto``` file.
The implementation of ```UpdateService``` holds the data handling/processing code. For code modularity I split those details into a ```simulate_game_update``` function which returns the processed ```UpdateRpcResponse``` which is then returned to the calling client through gRPC.
```rust
// berry-battler/src/main.rs
#[derive(Default)]
pub struct UpdateServiceTraitWrapper {}

#[tonic::async_trait]
impl UpdateService for UpdateServiceTraitWrapper {
    async fn update_rpc(
        &self,
        mut request: Request<UpdateRpcRequest>,
    ) -> Result<Response<UpdateRpcResponse>, Status> {
        println!("Got a request from {:?}", request.remote_addr());
        Ok(Response::new(simulate_game_update(request.get_mut())))
    }
}
```

We then have to start the server itself in ```main```. We let the variable ```update_rpc``` contain the default event for the ```UpdateServiceTraitWrapper``` which is simply our implementation of the ```UpdateService``` trait. I then pass that implementation to a server builder which creates the ```UpdateServiceServer```, starts it, and leaves it listening for client gRPC calls.
```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse().unwrap();

    println!("Server listening on {}", addr);

    let update_rpc = UpdateServiceTraitWrapper::default();
    Server::builder()
        .add_service(UpdateServiceServer::new(update_rpc))
        .serve(addr)
        .await?;

    Ok(())
}
```

### Data Handling
Preparing the data for transmission and interpreting the response is handled just like any normal Rust application. In this case the game simulation server (confusingly the gRPC client) prepares some simulated game data of different sizes and sends it to the AI node (the gRPC server). I will only show a portion of the source that demonstrates how the messages in the protocol definition translate into Rust ```structs``` that can be interacted with in the usual ways. Note that I moved these functions into tests since they are benchmarks and won't be part of the main application. The ```request_update``` function is in the main code body.
```rust
// berry-battle-server/src/main.rs
mod tests {
    use super::*;

    fn generate_sample_units(count: usize) -> Vec<UnitState> {
        let mut rng = thread_rng();

        let sample_unit = UnitState {
            id: rng.gen::<u32>(),
            unit_type: UnitType::Mobile as i32,
            position: Some(UnitPosition {
                x: rng.gen::<f32>(),
                y: rng.gen::<f32>(),
                layer: rng.gen::<u32>(),
                direction: Some(UnitDirectionVector {
                    x: rng.gen::<f32>(),
                    y: rng.gen::<f32>(),
                }),
            }),
            tag: "Sample".into(),
        };
        vec![sample_unit.clone(); count]
    }

    fn generate_update_state_request(
        unit_count: usize,
        processing_time_ns: u64,
    ) -> UpdateRpcRequest {
        let mut rng = thread_rng();
        UpdateRpcRequest {
            status: UpdateStatus::Processing as i32,
            update_id: rng.gen::<u32>(),
            units: generate_sample_units(unit_count),
            per_unit_proc_time_ns: processing_time_ns,
        }
    }
```

The AI node processes each "unit" like it would in a real game by modifying all the fields to random values and packaging them in a struct. I also have some timers in place to measure the elapsed time for data processing with some build in overhead, represented by a sleep timer, to simulate a more robust processing loop. This performance information is added to the the response data so it can be logged by the game simulation server. It is very similar to the code above and not much value to the discussion on gRPC.

I will note that in order to write tests that call ```async``` functions like ```request_update``` they will need to be adorned with the ``` #[tokio::test(flavor = "multi_thread")]``` attribute.

## Summary
Only portions of the code relating specifically to gRPC were detailed here but the entire project can be cloned, built, and executed for experimentation. 
[The AI Node](https://github.com/berrybattle/berry-battler/tree/v0.0.2)
[The Simulation Server](https://github.com/berrybattle/berry-battle-server/tree/v0.0.2)

By default, the source is configured to have both the AI node and the simulation server running on the same machine with a localhost connection. To run the code on separate machines, change "[::1]" to an IP address or host name.

The AI node should be started first with ```cargo run``` in either ```release``` or ```debug``` configurations.

Running the tests on the server can be done with ```cargo test --release -- --nocapture``` since the benchmarking code is wrapped in a test.

The output should look like this:
```console
Connecting to gRPC server
To simulate unit processing e.g. running A* etc on each unit
Apply constant processing time of 100000ns per unit

 Unit Count | Processing Time (us) | Latency (us) |
          0 |                    0 |         4303 |
         10 |                 1064 |         6020 |
         20 |                 2062 |         5174 |
         30 |                 3064 |         7636 |
         40 |                 4065 |         7991 |
         50 |                 5067 |         9615 |
         60 |                 6070 |        10414 |
         70 |                 7070 |        11162 |
         80 |                 8073 |        12808 |
         90 |                 9076 |        13594 |
        100 |                10076 |        14536 |
        110 |                11078 |        14693 |
        120 |                12076 |        15957 |
        130 |                13081 |        17615 |
        140 |                14080 |        18390 |
        150 |                15082 |        19572 |
        160 |                16084 |        20517 |
        170 |                17108 |        21769 |
        180 |                18091 |        22967 |
        190 |                19090 |        23434 |
        200 |                20091 |        24743 |
test tests::test_node_performance ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.66s
```



## Additional Resources
- [Compare gRPC services with HTTP APIs](https://learn.microsoft.com/en-us/aspnet/core/grpc/comparison?view=aspnetcore-8.0)
- [gRPC](https://grpc.io/)
- [Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/)
- [Crate tonic](https://docs.rs/tonic/latest/tonic/)
- [GitHub hyperium/tonic](https://github.com/hyperium/tonic)