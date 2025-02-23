📒 gpu-allocator
=

[![Actions Status](https://github.com/Traverse-Research/gpu-allocator/workflows/CI/badge.svg)](https://github.com/Traverse-Research/gpu-allocator/actions)
[![Latest version](https://img.shields.io/crates/v/gpu-allocator.svg)](https://crates.io/crates/gpu-allocator)
[![Docs](https://docs.rs/gpu-allocator/badge.svg)](https://docs.rs/gpu-allocator/)
[![LICENSE](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE-MIT)
[![LICENSE](https://img.shields.io/badge/license-apache-blue.svg)](LICENSE-APACHE)
[![Contributor Covenant](https://img.shields.io/badge/contributor%20covenant-v1.4%20adopted-ff69b4.svg)](../main/CODE_OF_CONDUCT.md)

[![Banner](banner.png)](https://traverseresearch.nl)

```toml
[dependencies]
gpu-allocator = "0.20.0"
```

This crate provides a fully written in Rust memory allocator for Vulkan and DirectX 12.

### [Windows-rs] and [winapi]

`gpu-allocator` recently migrated from [winapi] to [windows-rs] but still provides convenient helpers to convert to and from [winapi] types, enabled when compiling with the `public-winapi` crate feature.

[Windows-rs]: https://github.com/microsoft/windows-rs
[winapi]: https://github.com/retep998/winapi-rs

### Setting up the Vulkan memory allocator

```rust
use gpu_allocator::vulkan::*;

let mut allocator = Allocator::new(&AllocatorCreateDesc {
    instance,
    device,
    physical_device,
    debug_settings: Default::default(),
    buffer_device_address: true,  // Ideally, check the BufferDeviceAddressFeatures struct.
});
```

### Simple Vulkan allocation example

```rust
use gpu_allocator::vulkan::*;
use gpu_allocator::MemoryLocation;


// Setup vulkan info
let vk_info = vk::BufferCreateInfo::builder()
    .size(512)
    .usage(vk::BufferUsageFlags::STORAGE_BUFFER);

let buffer = unsafe { device.create_buffer(&vk_info, None) }.unwrap();
let requirements = unsafe { device.get_buffer_memory_requirements(buffer) };

let allocation = allocator
    .allocate(&AllocationCreateDesc {
        name: "Example allocation",
        requirements,
        location: MemoryLocation::CpuToGpu,
        linear: true, // Buffers are always linear
    }).unwrap();

// Bind memory to the buffer
unsafe { device.bind_buffer_memory(buffer, allocation.memory(), allocation.offset()).unwrap() };

// Cleanup
allocator.free(allocation).unwrap();
unsafe { device.destroy_buffer(buffer, None) };
```

### Setting up the D3D12 memory allocator

```rust
use gpu_allocator::d3d12::*;

let mut allocator = Allocator::new(&AllocatorCreateDesc {
    device,
    debug_settings: Default::default(),
});
```

### Simple d3d12 allocation example

```rust
use gpu_allocator::d3d12::*;
use gpu_allocator::MemoryLocation;


let buffer_desc = Direct3D12::D3D12_RESOURCE_DESC {
    Dimension: Direct3D12::D3D12_RESOURCE_DIMENSION_BUFFER,
    Alignment: 0,
    Width: 512,
    Height: 1,
    DepthOrArraySize: 1,
    MipLevels: 1,
    Format: Dxgi::Common::DXGI_FORMAT_UNKNOWN,
    SampleDesc: Dxgi::Common::DXGI_SAMPLE_DESC {
        Count: 1,
        Quality: 0,
    },
    Layout: Direct3D12::D3D12_TEXTURE_LAYOUT_ROW_MAJOR,
    Flags: Direct3D12::D3D12_RESOURCE_FLAG_NONE,
};
let allocation_desc = AllocationCreateDesc::from_d3d12_resource_desc(
    &allocator.device(),
    &buffer_desc,
    "Example allocation",
    MemoryLocation::GpuOnly,
);
let allocation = allocator.allocate(&allocation_desc).unwrap();
let mut resource: Option<Direct3D12::ID3D12Resource> = None;
let hr = unsafe {
    device.CreatePlacedResource(
        allocation.heap(),
        allocation.offset(),
        &buffer_desc,
        Direct3D12::D3D12_RESOURCE_STATE_COMMON,
        std::ptr::null(),
        &mut resource,
    )
}?;

// Cleanup
drop(resource);
allocator.free(allocation).unwrap();
```

### License

Licensed under either of

* Apache License, Version 2.0, ([LICENSE-APACHE](../master/LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](../master/LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Alternative libraries
* [vk-mem-rs](https://github.com/gwihlidal/vk-mem-rs)

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
