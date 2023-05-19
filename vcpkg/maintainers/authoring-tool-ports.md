---
title: Authoring Tool Ports
description: Guidelines on authoring tool ports for vcpkg.
ms.date: 05/16/2023
---
# Tool Ports

vcpkg is primarily a mechanism for delivering C++ libraries, but occasionally building libraries
requires access to tooling not intended to be distributed with the libraries. For example,
building a library that needs Meson needs Meson to be available on the system.

In vcpkg, we use 'tool ports' to represent these tools outside of a few exceptions. Tool ports
are [script ports](./authoring-script-ports.md)

The following tools have interaction outside of ports and should not be tool ports. If in doubt,
ask a maintainer!

* Tools that need to be present before vcpkg can invoke `portfile.cmake`.
* Compressors and decompressors like `tar` or `7z`.
* Tools that exist only to be binary caching providers like `aws`.

## Pattern / Requirements

### Do

* Provide a CMake function via [`vcpkg-port-config.cmake`](./authoring-script-ports.md) named
 `vcpkg_tool_<TOOLNAME>_find_acquire()` which returns the path to the tool.
* If a tool binary is downloaded, o the installed tree. Tools distributed in binary form
 should be located in their original downloads location.
* If tooling is broadly compatible such that the specific version returned rarely matters, as is
 the case for git, tool ports should try to use a copy of the tool for the system. However, if
 there is any doubt, a specific version locked downloaded copy should be used.
