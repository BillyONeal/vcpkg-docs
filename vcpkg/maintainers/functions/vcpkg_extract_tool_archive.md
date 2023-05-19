---
title: vcpkg_extract_tool_archive
description: Learn how to use vcpkg_extract_tool_archive.
ms.date: 05/18/2023
---
# vcpkg_extract_tool_archive

Extract an archive containing a tool.

## Usage

```cmake
vcpkg_extract_tool_archive(
    OUT_TOOL_PATH <tool_path_variable>
    ARCHIVE <path>
    TOOL_NAME <tool_name>
    VERSION <version>
    RELATIVE_PATH <relative_path>
)
```

## Parameters

### OUT_TOOL_PATH

Name of the variable to set with the full path to the tool. This will be the extract location of the `ARCHIVE`, combined with `RELATIVE_PATH`.

### ARCHIVE

Full path to the archive to extract.

### TOOL_NAME

The human visible name of the tool. For example, 'python3'.

### VERSION

The version of the tool downloaded. Different tool versions will be extracted into sibling directories.

### RELATIVE_PATH

The relative path within the `ARCHIVE` of the tool binary.

## Source

[scripts/cmake/vcpkg\_extract\_tool\_archive.cmake](https://github.com/Microsoft/vcpkg/blob/master/scripts/cmake/vcpkg_extract_tool_archive.cmake)
