+++
date = '2026-05-19T11:57:13-06:00'
draft = false
title = 'Building `vmvis`: Progress Report 00'
+++

I am starting work on a fun project I have been thinking about implementing for
some time. It is a visualizer for virtual memory on amd64/Linux. The past six
months or so I have been learning a lot about the ELF binary format and about
how virtual memory works on Linux so I think this would be a fun project to gain
even more experience with this sort of thing.

<iframe width="560" height="315" src="https://www.youtube.com/embed/cat6PXQXYxs?si=2B1CQqugRGCIcCn6" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Project Goals

My primary goals for this project are to:

- Show a zoomable visualization of the entire address space of any process
- Show all maps in `/proc/pid/maps`
- Allow for showing non-mapped space to give the user a sense of the scale of
  the virtual address space (it is big)
- Allow the user to zoom all the way down to a single byte of data
- Give a sense of scale for page, cache-line, and scalar sizes
- Provide an intuition for alignment and alignment issues

Secondary goals:

- Display section data from the binary
- Auto-disassemble instructions in .text sections
- Use dwarf data and other context to auto-interpret bytes for the user
- Visualize things like the Global Offset Table (GOT) and Procedure Link Table
  (PLT)
- Provide hand written explanations for different areas of memory for users to
  more easily follow along and learn
- Allow for the ability to select byte ranges and re-interpret data
- Visualize writes to memory in near-real-time
- Visualize each thread's execution context including instruction pointer, stack
  pointer, frame pointer, etc
- Visualize common memory allocator data structures if using libc malloc/free
- Support for huge pages
- Support for aarch64
- Support for Mach-O/macOS, likely not to happen but it would be helpful for
  non-linux users to learn about virtual memory

## Design and Implementation Ideas

I haven't put a ton of upfront thought into implementation but here are my
initial ideas which undoubtedly will be invalidated as I progress:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="/img/vmvis-00-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="/img/vmvis-00-light.svg">
  <img alt="Initial Design Ideas" src="/img/vmvis-00-dark.svg">
</picture>

The main design idea is to have a dedicated thread issuing updates to the screen
via `raylib`. `raylib` calls are single threaded so I'll need to find ways of
offloading work from that thread as needed to maintain UI performance.
Everything that can be done in separate threads to poll the process's state
should be done so.

Maps can easily be read from `/proc/pid/maps` periodically. `process_vm_readv`
is handy for quickly reading the process's memory without copying data through
the kernel.

`ptrace` can be used to attach to the running process. `ptrace` itself isn't
strictly needed to read maps or even to read memory from the process but
`ptrace` permissions are needed and `ptrace` does allow for an easy mechanism to
control the rate of execution and to read registers which will be needed for
visualizing execution context.
