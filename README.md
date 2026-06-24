# FsExt\-MCP\-Server\-Python

## Overview

**FsExt\-MCP\-Server\-Python** is a feature\-rich, secure, and high\-performance file system operation server implemented for the **Model Context Protocol \(MCP\)**\. It provides AI clients and MCP integrators with a complete set of file system, image processing, and OCR capabilities through standardized MCP tools\.

The server focuses on **security controllability** and **large\-file operation compatibility**, solving common pain points of basic MCP file tools such as insufficient permissions control, single function, and poor large\-file processing performance\.

### Core Features

- **Full File \& Directory Management**：Support file creation, deletion, copy, move, metadata query, existence check; full directory tree recursion copy and move\.

- **Flexible File Read/Write**：Integrate full text reading, segmented text reading, chunked binary reading, text/binary overwriting and appending writing, perfectly adapted to large file streaming operations\.

- **Powerful Search \& Replace**：Support directory\-wide file content search, single/multi\-file contextual matching \(with front/back line preview\), regular expression matching, case\-insensitive search, and in\-place text replacement\.

- **Image Processing Tools**：Built\-in image resize \(aspect ratio lock \& padding support\), crop, rotate, covering daily image editing demands\.

- **Tesseract OCR Recognition**：Support multi\-language text extraction from images, customizable Tesseract execution path and data path\.

- **Strict Workspace Security Lock**：Provide `--lock-root` directory restriction capability, all file/directory operations are limited to the specified root workspace to prevent unauthorized cross\-directory access\.

- **Multi\-Transport Support**：Compatible with official standard MCP transports: `stdio` \(local client integration\), `sse` \(lightweight remote stream\), `http` \(standard bidirectional remote streaming transport\)\.

## Installation

### Prerequisites

- Python ≥ 3\.10

- FFmpeg \(system global installation, required for media\-related dependencies\)

- Tesseract OCR \(optional, for image text extraction\)

### Install Dependencies

It is recommended to use `uv` for fast environment deployment:

```bash
# Clone repository
git clone https://github.com/kurtzhi/FsExt-Mcp-Server
cd FsExt-Mcp-Server

# Install all dependencies
uv sync

```

### Core Dependencies Description

- **chardet**：Automatic file character encoding detection

- **Pillow**：Core image processing library for resize, crop, rotate operations

- **python\-magic**：Accurate file MIME type detection

- **ffmpeg\-python**：FFmpeg Python wrapper for media processing

## Startup Usage

The server supports three MCP transport modes and flexible workspace root locking configuration\.

### Startup Parameters

|Parameter|Default Value|Description|
|---|---|---|
|`--transport`|stdio|MCP transport type: `stdio`/`sse`/`http` \(official standard transports\)|
|`--host`|127\.0\.0\.1|Bind address \(invalid under stdio mode\)|
|`--port`|8000|Bind port \(invalid under stdio mode\)|
|`--lock-root`|None|Lock all operations to the specified root directory \(unlimited full access if empty\)|

### Common Startup Commands

#### 1\. Default Local Stdio Mode \(for Claude Desktop / Cursor\)

```bash
uv run -m fsext

```

#### 2\. Stdio Mode with Workspace Lock \(Secure Local Use\)

```bash
uv run -m fsext --lock-root /your/workspace/path

```

#### 3\. Remote SSE Transport Mode

```bash
uv run -m fsext --transport sse --host 0.0.0.0 --port 8000
```

#### 4\. Official Standard HTTP Remote Transport \(Bidirectional Stream\)

```bash
uv run -m fsext --transport http --host 0.0.0.0 --port 8000
```

## Full MCP Tools List

All tools support **workspace path restriction** and follow standardized input/output specifications\.

### 1\. File Basic Operation Tools

#### fs\_create\_file

**Function**：Create a text file with optional initial content and custom encoding\.

**Parameters**：

- `file_path` \(str\): Target file path

- `content` \(str, default=""\): Initial text content

- `charset` \(str, default="utf\-8"\): File encoding format

**Returns**：`{"success": bool}`

#### fs\_delete\_file

**Function**：Permanently delete a single regular file \(directories are not supported\)\.

**Parameters**：`file_path` \(str\): Target file path

**Returns**：`{"success": bool}`

#### fs\_get\_file\_info

**Function**：Obtain complete metadata of files/directories \(type, size, modification time, permissions, etc\.\)\.

**Parameters**：`file_path` \(str\): Target path

**Returns**：Serialized full metadata dictionary of the target entry

#### fs\_is\_file\_exists

**Function**：Check whether the specified file or directory exists in the workspace\.

**Parameters**：`file_path` \(str\): Target path

**Returns**：`{"exists": bool}`

#### fs\_copy\_file

**Function**：Copy single file and retain original metadata, support overwrite switch\.

**Parameters**：

- `source_file_path` \(str\): Source file path

- `dest_file_path` \(str\): Target file path

- `overwrite` \(bool\): Whether to overwrite existing target file

**Returns**：`{"success": bool}`

#### fs\_move\_file

**Function**：Move single file to new path, support overwrite control\.

**Parameters**：

- `source_file_path` \(str\): Source file path

- `dest_file_path` \(str\): Target file path

- `overwrite` \(bool\): Whether to replace conflicting files

**Returns**：`{"success": bool}`

### 2\. Directory Operation Tools

#### fs\_list\_directory

**Function**：Scan directory and filter paths recursively, support file type filtering and pure file filtering\.

**Parameters**：

- `source_dir` \(str\): Target scan directory

- `recursive` \(bool\): Whether to scan subdirectories recursively

- `file_only` \(bool\): Whether to return only files \(exclude directories\)

- `file_extension` \(str, default=""\): Filter files by suffix

**Returns**：`{"paths": list[str]}` \(absolute path list\)

#### fs\_copy\_directory

**Function**：Recursively copy the entire directory tree, support overwriting existing directories\.

**Parameters**：

- `source_dir` \(str\): Source directory path

- `copy_dest_dir` \(str\): Target directory path

- `overwrite` \(bool\): Whether to clean and overwrite existing target directory

**Returns**：`{"success": bool}`

#### fs\_move\_directory

**Function**：Move the entire directory, fail actively if the target path exists \(avoid accidental overwriting\)\.

**Parameters**：

- `source_dir` \(str\): Source directory path

- `dest_dir` \(str\): Target directory path

**Returns**：`{"success": bool}`

### 3\. File Read \& Write Tools

#### fs\_read\_full\_text

**Function**：Read full content of text file and count total lines\.

**Parameters**：

- `file_path` \(str\): Target file path

- `charset` \(str, default="utf\-8"\): File encoding

**Returns**：`{"n_lines": int, "content": str}`

#### fs\_read\_text\_range

**Function**：Read segmented text content, support skipping lines and limiting reading lines \(adapt to large text files\)\.

**Parameters**：

- `file_path` \(str\): Target file path

- `lines_to_skip` \(int, default=0\): Skip leading lines

- `max_lines_to_read` \(int, default=0\): Limit read lines \(\-1 for unlimited\)

- `line_separator` \(str, default="\\n"\): Line break separator

- `charset` \(str, default="utf\-8"\): File encoding

**Returns**：`{"n_lines": int, "content": str}`

#### fs\_read\_binary\_chunk

**Function**：Chunked reading of binary files, return Base64 encoded data for safe transmission\.

**Parameters**：

- `file_path` \(str\): Target file path

- `bytes_to_skip` \(int, default=0\): Skip leading bytes

- `max_bytes_to_read` \(int, default=0\): Limit read bytes \(\-1 for unlimited\)

**Returns**：`{"actual_length": int, "end_of_stream": bool, "data_base64": str}`

#### fs\_write\_text

**Function**：Write text content, support overwrite and append modes\.

**Parameters**：

- `file_path` \(str\): Target file path

- `text` \(str\): Text content to write

- `append` \(bool, default=False\): Append mode switch

**Returns**：`{"success": bool}`

#### fs\_write\_binary

**Function**：Segmented binary writing, support offset slicing and appending\.

**Parameters**：

- `file_path` \(str\): Target file path

- `data` \(bytes\): Binary data to write

- `offset` \(int, default=0\): Data offset

- `length` \(int, default=\-1\): Write data length \(\-1 for full buffer\)

- `append` \(bool, default=False\): Append mode switch

**Returns**：`{"success": bool}`

### 4\. Search \& Replace Tools

#### fs\_search\_files\_by\_content

**Function**：Scan directory to find all files containing target content, support regex, case insensitivity and file type filtering\.

**Parameters**：

- `dir_path` \(str\): Scan root directory

- `recursive` \(bool\): Recursive scan switch

- `search_term` \(str\): Search keyword / regex pattern

- `is_regex` \(bool, default=False\): Whether to enable regex matching

- `ignore_case` \(bool, default=True\): Case\-insensitive matching

- `file_type` \(str, default=""\): Filter file suffix

- `charset` \(str, default="utf\-8"\): File encoding

**Returns**：`{"file_paths": list[str]}`

#### fs\_search\_in\_files\_by\_content

**Function**：Multi\-file content matching, return matching segments with customizable front/back context lines and result limit\.

**Parameters**：

- `dir_path` \(str\): Scan root directory

- `recursive` \(bool\): Recursive scan switch

- `search_term` \(str\): Search keyword / regex pattern

- `is_regex` \(bool\): Regex enable switch

- `ignore_case` \(bool\): Case\-insensitive switch

- `limit` \(int\): Maximum matching result count

- `lines_before` \(int\): Preceding context lines

- `lines_after` \(int\): Subsequent context lines

- `file_extension` \(str, default=""\): File suffix filter

- `charset` \(str, default="utf\-8"\): File encoding

**Returns**：Structured list of matching results with file path and line context

#### fs\_search\_in\_file\_by\_content

**Function**：Precise content search for single file, return matching content with line context\.

**Parameters**：

- `file_path` \(str\): Target file path

- `search_term` \(str\): Search keyword / regex pattern

- `is_regex` \(bool\): Regex enable switch

- `ignore_case` \(bool\): Case\-insensitive switch

- `lines_before` \(int\): Preceding context lines

- `lines_after` \(int\): Subsequent context lines

- `charset` \(str, default="utf\-8"\): File encoding

**Returns**：Structured list of single\-file matching results

#### fs\_file\_replace

**Function**：In\-place text replacement for single file, return modified line count\.

**Parameters**：

- `file_path` \(str\): Target file path

- `search_term` \(str\): Content to be replaced

- `replacement` \(str\): New replacement content

- `line_separator` \(str, default="\\n"\): Line break separator

**Returns**：`{"replaced_line_count": int}`

### 5\. Image Processing \& OCR Tools

#### fs\_image\_resize

**Function**：Resize image to specified size, support aspect ratio locking and canvas padding\.

**Parameters**：

- `source_image_path` \(str\): Source image path

- `dest_image_path` \(str\): Output image path

- `keep_aspect_ratio` \(bool\): Lock original image aspect ratio

- `pad_to_target` \(bool\): Fill blank with padding to fit target size

- `target_width` \(int\): Target image width

- `target_height` \(int\): Target image height

**Returns**：`{"success": bool}`

#### fs\_image\_crop

**Function**：Crop rectangular area from source image and export new image file\.

**Parameters**：

- `source_image_path` \(str\): Source image path

- `dest_image_path` \(str\): Output image path

- `x` \(int\): Crop start X coordinate

- `y` \(int\): Crop start Y coordinate

- `width` \(int\): Crop area width

- `height` \(int\): Crop area height

**Returns**：`{"success": bool}`

#### fs\_image\_rotate

**Function**：Rotate image clockwise, automatically expand canvas to retain full image content\.

**Parameters**：

- `source_image_path` \(str\): Source image path

- `dest_image_path` \(str\): Output image path

- `degrees` \(float\): Clockwise rotation angle

**Returns**：`{"success": bool}`

#### fs\_ocr\_extract\_text

**Function**：Extract text from images based on Tesseract OCR engine, support multi\-language recognition\.

**Parameters**：

- `image_path` \(str\): Target image path

- `tesseract_cmd_path` \(str\): Tesseract executable path

- `lang` \(str\): Recognition language code

- `tessdata_path` \(str, default=""\): Tesseract language data path

**Returns**：`{"ocr_text": str}`

## License

This project is open\-sourced under the **Apache License 2\.0**\. See the `LICENSE` file in the project root for full license details\.

## Third\-Party Licenses

This project depends on open\-source libraries including **chardet**, **Pillow**, **python\-magic**, **ffmpeg\-python**\. All third\-party components comply with their respective open\-source license specifications\.

Note: FFmpeg binaries are not bundled in this project; please follow FFmpeg's official license terms when using it\.

## Contact \& Repository

GitHub: [https://github\.com/kurtzhi/fsext\-mcp\-server\-python](https://github.com/kurtzhi/fsext-mcp-Server-python)
