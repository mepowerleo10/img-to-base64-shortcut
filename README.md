# img-to-base64-shortcut

A PowerShell and Bash/Zsh shortcut for copying an image's Base64 representation to the clipboard. It supports image files passed by path, image files copied in the file manager, and raw images in the clipboard. Add `-l` or `--link` / `-DataUri` to include a `data:<mime-type>;base64,` prefix.

## Table of Contents

- [Windows](#windows)
  - [Setup](#setup)
  - [Function](#function)
  - [Usage](#usage)
- [Linux](#linux)
  - [Prerequisites](#prerequisites)
  - [Setup](#setup-1)
  - [Function](#function-1)
  - [Usage](#usage-1)

## Windows

### Setup

1. Open your PowerShell profile in Notepad, creating it first if it does not exist:

   ```powershell
   if (!(Test-Path -Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
   notepad $PROFILE
   ```

2. Paste the function and alias below into the profile, then save and close Notepad.
3. Load the updated profile in the current PowerShell session:

   ```powershell
   . $PROFILE
   ```

### Function

```powershell
function ConvertTo-ImageBase64 {
    param(
        [Parameter(Position = 0)]
        [string]$Path,

        [Alias("l")]
        [switch]$DataUri
    )

    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    # Helper to format output with optional Data URI prefix
    function Format-Result ([byte[]]$Bytes, [string]$MimeType) {
        $b64 = [Convert]::ToBase64String($Bytes)
        if ($DataUri) { return "data:$MimeType;base64,$b64" }
        return $b64
    }

    # Helper to map file extensions to MIME types
    function Get-MimeType ([string]$FilePath) {
        switch ([System.IO.Path]::GetExtension($FilePath).ToLower()) {
            ".png"  { "image/png" }
            ".jpg"  { "image/jpeg" }
            ".jpeg" { "image/jpeg" }
            ".gif"  { "image/gif" }
            ".webp" { "image/webp" }
            ".bmp"  { "image/bmp" }
            ".svg"  { "image/svg+xml" }
            default { "application/octet-stream" }
        }
    }

    # 1. File path passed as an argument
    if ($Path) {
        $resolved = (Resolve-Path $Path -ErrorAction Stop).Path
        $bytes = [System.IO.File]::ReadAllBytes($resolved)
        Format-Result $bytes (Get-MimeType $resolved) | Set-Clipboard
        Write-Host "Copied Base64 from file: $resolved"
        return
    }

    # 2. File copied in File Explorer (preserves original JPG/PNG/WebP bytes)
    $files = Get-Clipboard -Format FileDropList
    if ($files -and $files.Count -gt 0) {
        $filePath = $files[0].FullName
        $bytes = [System.IO.File]::ReadAllBytes($filePath)
        Format-Result $bytes (Get-MimeType $filePath) | Set-Clipboard
        Write-Host "Copied Base64 from clipboard file: $filePath"
        return
    }

    # 3. Raw bitmap in clipboard (screenshots / "Copy Image")
    $img = [System.Windows.Forms.Clipboard]::GetImage()
    if ($img) {
        $ms = New-Object System.IO.MemoryStream
        $img.Save($ms, [System.Drawing.Imaging.ImageFormat]::Png)
        Format-Result $ms.ToArray() "image/png" | Set-Clipboard
        $ms.Dispose()
        $img.Dispose()
        Write-Host "Copied Base64 from clipboard image (encoded as PNG)!"
        return
    }

    Write-Warning "No file path provided and no image or file found in clipboard."
}

Set-Alias -Name img64 -Value ConvertTo-ImageBase64
```

### Usage

```powershell
img64 C:\path\to\image.png
img64 C:\path\to\image.png -DataUri
img64 -l
```

Copy an image file in File Explorer or copy a screenshot directly, then run `img64` to place the Base64 string or Data URI on the clipboard.

## Linux

### Prerequisites

Install the clipboard utility for your display server (`file` is usually pre-installed and is used for MIME detection):

- **Wayland:** [`wl-clipboard`](https://github.com/bugaevc/wl-clipboard), which provides `wl-copy` and `wl-paste`
- **X11:** `xclip`

### Setup

Add the function below to `~/.bashrc` or `~/.zshrc`, then reload the shell configuration:

```bash
source ~/.bashrc
```

For Zsh:

```bash
source ~/.zshrc
```

### Function

```bash
img64() {
    local as_link=0
    local filepath=""

    # Parse arguments (-l flag and optional file path)
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -l|--link)
                as_link=1
                shift
                ;;
            *)
                filepath="$1"
                shift
                ;;
        esac
    done

    # Helper to copy output to Wayland or X11 clipboard
    _copy_clip() {
        if [[ -n "$WAYLAND_DISPLAY" ]] && command -v wl-copy &>/dev/null; then
            wl-copy
        elif [[ -n "$DISPLAY" ]] && command -v xclip &>/dev/null; then
            xclip -selection clipboard
        else
            echo "Error: Neither wl-copy (Wayland) nor xclip (X11) found." >&2
            return 1
        fi
    }

    # Helper to encode stream and optionally prepend data URI
    _encode_and_copy() {
        local mime="$1"
        local b64
        b64=$(base64 -w 0)
        if [[ $as_link -eq 1 ]]; then
            printf "data:%s;base64,%s" "$mime" "$b64" | _copy_clip
        else
            printf "%s" "$b64" | _copy_clip
        fi
    }

    # 1. File path passed as an argument
    if [[ -n "$filepath" ]]; then
        if [[ ! -f "$filepath" ]]; then
            echo "Error: File not found: $filepath" >&2
            return 1
        fi
        local mime
        mime=$(file --mime-type -b "$filepath")
        _encode_and_copy "$mime" < "$filepath"
        echo "Copied Base64 from file: $filepath ($mime)"
        return 0
    fi

    # 2. Wayland clipboard handling
    if [[ -n "$WAYLAND_DISPLAY" ]] && command -v wl-paste &>/dev/null; then
        local types
        types=$(wl-paste --list-types 2>/dev/null)

        # Check if a file was copied in a file manager (text/uri-list)
        if echo "$types" | grep -q "^text/uri-list$"; then
            local uri
            uri=$(wl-paste --type text/uri-list | head -n 1 | tr -d '\r')
            if [[ "$uri" == file://* ]]; then
                local decoded_path
                decoded_path=$(python3 -c "import sys, urllib.parse; print(urllib.parse.unquote(sys.argv[1][7:]))" "$uri")
                if [[ -f "$decoded_path" ]]; then
                    local mime
                    mime=$(file --mime-type -b "$decoded_path")
                    _encode_and_copy "$mime" < "$decoded_path"
                    echo "Copied Base64 from clipboard file: $decoded_path ($mime)"
                    return 0
                fi
            fi
        fi

        # Check for raw image data in clipboard (prefer PNG, fallback to JPEG/WebP)
        for img_mime in image/png image/jpeg image/webp; do
            if echo "$types" | grep -q "^${img_mime}$"; then
                wl-paste --type "$img_mime" | _encode_and_copy "$img_mime"
                echo "Copied Base64 from Wayland clipboard ($img_mime)!"
                return 0
            fi
        done

    # 3. X11 clipboard handling
    elif [[ -n "$DISPLAY" ]] && command -v xclip &>/dev/null; then
        local targets
        targets=$(xclip -selection clipboard -t TARGETS -o 2>/dev/null)

        # Check if a file was copied in a file manager
        if echo "$targets" | grep -q "^text/uri-list$"; then
            local uri
            uri=$(xclip -selection clipboard -t text/uri-list -o 2>/dev/null | head -n 1 | tr -d '\r')
            if [[ "$uri" == file://* ]]; then
                local decoded_path
                decoded_path=$(python3 -c "import sys, urllib.parse; print(urllib.parse.unquote(sys.argv[1][7:]))" "$uri")
                if [[ -f "$decoded_path" ]]; then
                    local mime
                    mime=$(file --mime-type -b "$decoded_path")
                    _encode_and_copy "$mime" < "$decoded_path"
                    echo "Copied Base64 from clipboard file: $decoded_path ($mime)"
                    return 0
                fi
            fi
        fi

        # Check for raw image data in clipboard
        for img_mime in image/png image/jpeg image/webp; do
            if echo "$targets" | grep -q "^${img_mime}$"; then
                xclip -selection clipboard -t "$img_mime" -o | _encode_and_copy "$img_mime"
                echo "Copied Base64 from X11 clipboard ($img_mime)!"
                return 0
            fi
        done
    fi

    echo "Warning: No file path provided and no image or file found in clipboard." >&2
    return 1
}
```

### Usage

```bash
img64                              # From a screenshot or copied image
img64 -l                           # From a screenshot or copied image as a Data URI
img64 ~/Pictures/photo.jpg         # From a file path
img64 -l ~/Pictures/photo.jpg      # From a file path as a Data URI
```

The `base64 -w 0` option prevents GNU `base64` from inserting line breaks every 76 characters, matching the single-line PowerShell output.
