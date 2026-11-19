# img-to-base64-shortcut

A PowerShell function and `img64` alias for copying an image's Base64 representation to the clipboard. It can read an image file by path, use an image file copied in File Explorer, or encode a raw image from the clipboard as PNG. Use the `-DataUri` (or `-l`) switch to include a `data:<mime-type>;base64,` prefix.

## Setup

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

You can now run `img64 C:\path\to\image.png`, or copy an image file/image to the clipboard and run `img64`. Add `-DataUri` to get a Data URI instead of plain Base64, for example `img64 C:\path\to\image.png -DataUri`.

## Function

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
