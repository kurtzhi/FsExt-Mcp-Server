# FsExt-MCP-Server (Python)

## Overview

**FsExt-MCP-Server (Python)** is a feature-rich, secure, and high-performance file system operation server implemented for the **Model Context Protocol (MCP)**. It provides AI clients and MCP integrators with a complete set of file system, image processing, and OCR capabilities through standardized MCP tools.

The server focuses on **security controllability** and **large-file operation compatibility**, solving common pain points of basic MCP file tools such as insufficient permissions control, single function, and poor large-file processing performance.

### Core Features

- **Full File & Directory Management**: Support file creation, deletion, copy, move, metadata query, existence check; full directory tree recursion copy and move.

- **Flexible File Read/Write**: Integrate full text reading, segmented text reading, chunked binary reading, text/binary overwriting and appending writing, perfectly adapted to large file streaming operations.

- **Powerful Search & Replace**: Support directory-wide file content search, single/multi-file contextual matching (with front/back line preview), regular expression matching, case-insensitive search, and in-place text replacement.

- **Image Processing Tools**: Built-in image resize (aspect ratio lock & padding support), crop, rotate, covering daily image editing demands.

- **Tesseract OCR Recognition**: Support multi-language text extraction from images, customizable Tesseract execution path and data path.

- **Strict Workspace Security Lock**: Provide `--lock-root` directory restriction capability, all file/directory operations are limited to the specified root workspace to prevent unauthorized cross-directory access.

- **Multi-Transport Support**: Compatible with official standard MCP transports: `stdio` (local client integration), `sse` (lightweight remote stream), `http` (standard bidirectional remote streaming transport).

## Installation

### Prerequisites

- Python ≥ 3.10
- FFmpeg (system global installation, required for media-related dependencies)
- Tesseract OCR (optional, for image text extraction)

### Install Dependencies

It is recommended to use `uv` for fast environment deployment:

```bash
# Clone repository
git clone https://github.com/kurtzhi/fsext-mcp-server-python
cd fsext-mcp-server-python

# Install all dependencies
uv sync
```

### Core Dependencies Description

- **chardet**: Automatic file character encoding detection
- **Pillow**: Core image processing library for resize, crop, rotate operations
- **python-magic**: Accurate file MIME type detection
- **ffmpeg-python**: FFmpeg Python wrapper for media processing

## Startup Usage

The server supports three MCP transport modes and flexible workspace root locking configuration.

### Startup Parameters

| Parameter | Default Value | Description |
| --- | --- | --- |
| `--transport` | stdio | MCP transport type: `stdio`/`sse`/`http` (official standard transports) |
| `--host` | 127.0.0.1 | Bind address (invalid under stdio mode) |
| `--port` | 8000 | Bind port (invalid under stdio mode) |
| `--lock-root` | None | Lock all operations to the specified root directory (unlimited full access if empty) |

### Common Startup Commands

#### 1. Default Local Stdio Mode (for Claude Desktop / Cursor)
```bash
uv run -m fsext
```

#### 2. Stdio Mode with Workspace Lock (Secure Local Use)
```bash
uv run -m fsext --lock-root /your/workspace/path
```

#### 3. Remote SSE Transport Mode
```bash
uv run -m fsext --transport sse --host 0.0.0.0 --port 8000
```
##### Access Endpoints
1. SSE long stream receive channel (client subscribe server push messages)
   `http://<host>:<port>/sse`
2. Client request send channel (call tools, send initialize request)
   `http://<host>:<port>/messages`

##### Client Config (MCP Inspector)
- Transport type: SSE
- Connection address fill: `http://127.0.0.1:8000/sse`

#### 4. Official Standard HTTP Remote Transport (Bidirectional Stream)
```bash
uv run -m fsext --transport http --host 0.0.0.0 --port 8000
```
##### Access Endpoint
Only single unified bidirectional entry point, no separate /sse or /messages routes:
`http://<host>:<port>/mcp`

##### Client Config (MCP Inspector)
- Transport type: Streamable HTTP
- Connection address fill: `http://127.0.0.1:8000/mcp`

#### 5. Key Differences between SSE Transport & Streamable HTTP Transport

| Feature | SSE (HTTP+SSE) | Streamable HTTP |
| --- | --- | --- |
| **Endpoint Architecture** | **Two endpoints**: Requires a POST endpoint for client requests and a GET endpoint for server streams. | **Single endpoint**: Uses the same URL/path for both POST requests and GET streams. |
| **Communication** | **Unidirectional**: Server pushes events, but clients cannot easily send data back through that same stream. | **Bidirectional (optional)**: Handles normal HTTP responses or seamlessly enables SSE for real-time notifications on the same connection. |
| **Connection Stability** | Prone to session dropouts and connection-loss issues due to complex state management across two endpoints. | Automatically recovers sessions and performs better under high concurrency with lower maintenance costs. |
| **Current Standard Status** | Deprecated for modern distributed applications. | The modern standard for remote client-server integrations. |

## Full MCP Tools List
### Unified Global Response Specification
All MCP tools share a unified top-level wrapping structure for both successful execution and runtime errors. Every tool’s raw business return data is placed inside the `info` field under `res`.

#### Structure Definition
```json
{
  "res": {
    "success": boolean,
    "info": object
  }
}
```
- `success`: Global execution flag
    - `true`: Tool runs normally, `info` carries business return data defined in each tool’s Returns section
    - `false`: Execution failed (workspace access restriction, missing file, IO failure, invalid parameters, etc.)
- `info` dual modes:
    1. Success mode (`success: true`): Custom business data object unique to the tool
    2. Error mode (`success: false`): Fixed error object with error code and human-readable message
       ```json
       "info": {
         "code": "ERROR_CODE",
         "message": "Detailed error description"
       }
       ```

#### Full Example Responses
##### 1. Successful Response Example (fs_list_directory)
```json
{
  "res": {
    "success": true,
    "info": {
      "paths": [
        "/tmp/tests/test_util.py",
        "/tmp/tests/__init__.py",
        "/tmp/tests/img/cochem_castle.jpg"
      ]
    }
  }
}
```

##### 2. Failed Response Example (Workspace Restriction Block)
```json
{
  "res": {
    "success": false,
    "info": {
      "code": "WORKSPACE_ESCAPE_FORBIDDEN",
      "message": "Access restricted: Path `/tmp/test2` is outside allowed workspace `/tmp/tests`"
    }
  }
}
```

All tools support **workspace path restriction** and follow standardized input/output specifications.

### 1. File Basic Operation Tools
#### fs_create_file
**Function**: Create a text file with optional initial content and custom encoding.

**Parameters**:
- `file_path` (str): Target file path
- `content` (str, default=""): Initial text content
- `charset` (str, default="utf-8"): File encoding format

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_delete_file
**Function**: Permanently delete a single regular file (directories are not supported).

**Parameters**: `file_path` (str): Target file path

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_get_file_info
**Function**: Obtain complete metadata of files/directories (type, size, modification time, permissions, etc.).

**Parameters**: `file_path` (str): Target path

**Raw Business Returns**: Serialized full metadata dictionary of the target entry
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_is_file_exists
**Function**: Check whether the specified file or directory exists in the workspace.

**Parameters**: `file_path` (str): Target path

**Raw Business Returns**: `{"exists": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_copy_file
**Function**: Copy single file and retain original metadata, support overwrite switch.

**Parameters**:
- `source_file_path` (str): Source file path
- `dest_file_path` (str): Target file path
- `overwrite` (boolean): Whether to overwrite existing target file

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_move_file
**Function**: Move single file to new path, support overwrite control.

**Parameters**:
- `source_file_path` (str): Source file path
- `dest_file_path` (str): Target file path
- `overwrite` (boolean): Whether to replace conflicting files

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

### 2. Directory Operation Tools
#### fs_list_directory
**Function**: Scan directory and filter paths recursively, support file type filtering and pure file filtering.

**Parameters**:
- `source_dir` (str): Target scan directory
- `recursive` (boolean): Whether to scan subdirectories recursively
- `file_only` (boolean): Whether to return only files (exclude directories)
- `file_extension` (str, default=""): Filter files by suffix

**Raw Business Returns**: `{"paths": list[str]}` (absolute path list)
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_copy_directory
**Function**: Recursively copy the entire directory tree, support overwriting existing directories.

**Parameters**:
- `source_dir` (str): Source directory path
- `copy_dest_dir` (str): Target directory path
- `overwrite` (boolean): Whether to clean and overwrite existing target directory

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_move_directory
**Function**: Move the entire directory, fail actively if the target path exists (avoid accidental overwriting).

**Parameters**:
- `source_dir` (str): Source directory path
- `dest_dir` (str): Target directory path

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

### 3. File Read & Write Tools
#### fs_read_full_text
**Function**: Read full content of text file and count total lines.

**Parameters**:
- `file_path` (str): Target file path
- `charset` (str, default="utf-8"): File encoding

**Raw Business Returns**: `{"n_lines": int, "content": str}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_read_text_range
**Function**: Read segmented text content, support skipping lines and limiting reading lines (adapt to large text files).

**Parameters**:
- `file_path` (str): Target file path
- `lines_to_skip` (int, default=0): Skip leading lines
- `max_lines_to_read` (int, default=0): Limit read lines (-1 for unlimited)
- `line_separator` (str, default="\n"): Line break separator
- `charset` (str, default="utf-8"): File encoding

**Raw Business Returns**: `{"n_lines": int, "content": str}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_read_binary_chunk
**Function**: Chunked reading of binary files, return Base64 encoded data for safe transmission.

**Parameters**:
- `file_path` (str): Target file path
- `bytes_to_skip` (int, default=0): Skip leading bytes
- `max_bytes_to_read` (int, default=0): Limit read bytes (-1 for unlimited)

**Raw Business Returns**: `{"actual_length": int, "end_of_stream": boolean, "data_base64": str}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_write_text
**Function**: Write text content, support overwrite and append modes.

**Parameters**:
- `file_path` (str): Target file path
- `text` (str): Text content to write
- `append` (boolean, default=False): Append mode switch

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_write_binary
**Function**: Segmented binary writing, support offset slicing and appending.

**Parameters**:
- `file_path` (str): Target file path
- `data` (bytes): Binary data to write
- `offset` (int, default=0): Data offset
- `length` (int, default=-1): Write data length (-1 for full buffer)
- `append` (boolean, default=False): Append mode switch

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

### 4. Search & Replace Tools
#### fs_search_files_by_content
**Function**: Scan directory to find all files containing target content, support regex, case insensitivity and file type filtering.

**Parameters**:
- `dir_path` (str): Scan root directory
- `recursive` (boolean): Recursive scan switch
- `search_term` (str): Search keyword / regex pattern
- `is_regex` (boolean, default=False): Whether to enable regex matching
- `ignore_case` (boolean, default=True): Case-insensitive matching
- `file_type` (str, default=""): Filter file suffix
- `charset` (str, default="utf-8"): File encoding

**Raw Business Returns**: `{"file_paths": list[str]}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_search_in_files_by_content
**Function**: Multi-file content matching, return matching segments with customizable front/back context lines and result limit.

**Parameters**:
- `dir_path` (str): Scan root directory
- `recursive` (boolean): Recursive scan switch
- `search_term` (str): Search keyword / regex pattern
- `is_regex` (boolean): Regex enable switch
- `ignore_case` (boolean): Case-insensitive switch
- `limit` (int): Maximum matching result count
- `lines_before` (int): Preceding context lines
- `lines_after` (int): Subsequent context lines
- `file_extension` (str, default=""): File suffix filter
- `charset` (str, default="utf-8"): File encoding

**Raw Business Returns**: Structured list of matching results with file path and line context
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_search_in_file_by_content
**Function**: Precise content search for single file, return matching content with line context.

**Parameters**:
- `file_path` (str): Target file path
- `search_term` (str): Search keyword / regex pattern
- `is_regex` (boolean): Regex enable switch
- `ignore_case` (boolean): Case-insensitive switch
- `lines_before` (int): Preceding context lines
- `lines_after` (int): Subsequent context lines
- `charset` (str, default="utf-8"): File encoding

**Raw Business Returns**: Structured list of single-file matching results
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_file_replace
**Function**: In-place text replacement for single file, return modified line count.

**Parameters**:
- `file_path` (str): Target file path
- `search_term` (str): Content to be replaced
- `replacement` (str): New replacement content
- `line_separator` (str, default="\n"): Line break separator

**Raw Business Returns**: `{"replaced_line_count": int}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

### 5. Image Processing & OCR Tools
#### fs_image_resize
**Function**: Resize image to specified size, support aspect ratio locking and canvas padding.

**Parameters**:
- `source_image_path` (str): Source image path
- `dest_image_path` (str): Output image path
- `keep_aspect_ratio` (boolean): Lock original image aspect ratio
- `pad_to_target` (boolean): Fill blank with padding to fit target size
- `target_width` (int): Target image width
- `target_height` (int): Target image height

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_image_crop
**Function**: Crop rectangular area from source image and export new image file.

**Parameters**:
- `source_image_path` (str): Source image path
- `dest_image_path` (str): Output image path
- `x` (int): Crop start X coordinate
- `y` (int): Crop start Y coordinate
- `width` (int): Crop area width
- `height` (int): Crop area height

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_image_rotate
**Function**: Rotate image clockwise, automatically expand canvas to retain full image content.

**Parameters**:
- `source_image_path` (str): Source image path
- `dest_image_path` (str): Output image path
- `degrees` (float): Clockwise rotation angle

**Raw Business Returns**: `{"success": boolean}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

#### fs_ocr_extract_text
**Function**: Extract text from images based on Tesseract OCR engine, support multi-language recognition.

**Parameters**:
- `image_path` (str): Target image path
- `tesseract_cmd_path` (str): Tesseract executable path
- `lang` (str): Recognition language code
- `tessdata_path` (str, default=""): Tesseract language data path

**Raw Business Returns**: `{"ocr_text": str}`
> The above raw data will be wrapped in the unified global `res` structure shown in the specification above.

## License

This project is open-sourced under the **Apache License 2.0**. See the `LICENSE` file in the project root for full license details.

## Third-Party Licenses

This project depends on open-source libraries including **chardet**, **Pillow**, **python-magic**, **ffmpeg-python**. All third-party components comply with their respective open-source license specifications.

Note: FFmpeg binaries are not bundled in this project; please follow FFmpeg's official license terms when using it.

## Contact & Repository

GitHub: [https://github.com/kurtzhi/fsext-mcp-server-python](https://github.com/kurtzhi/fsext-mcp-server-python)