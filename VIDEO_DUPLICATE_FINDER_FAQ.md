# Video Duplicate Finder - FAQ & Technical Details

## 1. Scanning Drives Separately - Finding Duplicates Without Both Physical Files

**YES, this is fully supported!** There are two key settings in `VDF.Core/Settings.cs`:

```csharp
public bool IncludeNonExistingFiles = true;      // DEFAULT: ON
public bool ScanAgainstEntireDatabase;           // Compare against ALL indexed files
```

### Workflow for Multiple External Drives

1. **Plug in Drive A** → Scan → Fingerprints saved to `ScannedFiles.db`
2. **Unplug Drive A, plug in Drive B** → Scan with `ScanAgainstEntireDatabase = true`
3. **Comparison runs against entire database** - including Drive A's fingerprints (even though files are offline)

### Database Portability

The tool supports **JSON export/import** (`VDF.Core/Utils/DatabaseUtils.cs:134-165`) so you can:
- Export database from one machine
- Import on another
- Merge fingerprint databases from different sources

### Key Settings for Offline Comparison

| Setting | Default | Purpose |
|---------|---------|---------|
| `IncludeNonExistingFiles` | `true` | Include files that no longer exist in comparisons |
| `ScanAgainstEntireDatabase` | `false` | When enabled, compares against ALL indexed files, not just current scan |

---

## 2. Scan Level vs Comparison Level Parameters

Parameters are divided into two phases:

### Scan-Level Parameters (Frame Extraction)

These affect how fingerprints are generated. **Changing these requires re-scanning.**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ThumbnailCount` | 1 | Number of frames to extract per video |
| `HardwareAccelerationMode` | Auto | FFmpeg GPU decoding (CUDA, VAAPI, etc.) |
| `UseNativeFfmpegBinding` | false | Use native FFmpeg bindings vs subprocess |
| `MaxDegreeOfParallelism` | 1 | Parallel file processing |
| `AlwaysRetryFailedSampling` | false | Retry files that previously failed |

### Comparison-Level Parameters

These affect how fingerprints are compared. **Can be changed without re-scanning.**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `Percent` | 96% | Similarity threshold |
| `PercentDurationDifference` | 20% | Duration tolerance (±20%) |
| `UsePHashing` | false | Use pHash instead of graybytes |
| `IgnoreBlackPixels` | false | Ignore letterboxing/black borders |
| `IgnoreWhitePixels` | false | Ignore watermarks/white overlays |
| `CompareHorizontallyFlipped` | false | Detect mirrored videos |
| `IncludeNonExistingFiles` | true | Include offline files in comparison |
| `ScanAgainstEntireDatabase` | false | Compare against all indexed files |
| `ExcludeHardLinks` | false | Skip hardlinked files |

### Re-running Comparison Without Re-scanning

The `StartCompare()` method in `VDF.Core/ScanEngine.cs:113` runs comparison independently:

```csharp
public async void StartCompare() {
    // Runs comparison on existing database without re-extracting frames
}
```

This means you can tweak comparison parameters and re-run quickly.

---

## 3. How It Handles Different Durations & Watermarks

### Different Video Durations

The tool uses percentage-based duration tolerance:

```csharp
// VDF.Core/ScanEngine.cs:484-485
double maxPercentDurationDifference = 100d + Settings.PercentDurationDifference;  // 120%
double minPercentDurationDifference = 100d - Settings.PercentDurationDifference;  // 80%

// VDF.Core/ScanEngine.cs:505-508
double p = entry.Duration / compItem.Duration * 100d;
if (p > maxPercentDurationDifference || p < minPercentDurationDifference)
    continue;  // Skip - too different in length
```

**Example with default 20% tolerance:**
- A 100-second video matches videos from **80s to 125s**
- A 60-minute movie matches movies from **48 to 75 minutes**

### Watermark Handling

Two pixel-filtering options handle overlays:

```csharp
// VDF.Core/Settings.cs:32-33
public bool IgnoreBlackPixels;   // Pixels < 0x20 (letterboxing, black borders)
public bool IgnoreWhitePixels;   // Pixels > 0xF0 (white text, watermarks)
```

**How it works:**
- When `IgnoreWhitePixels = true`, pixels with brightness > 240 (0xF0) are excluded from comparison
- A video with a white logo watermark will still match the clean original
- Similarly, `IgnoreBlackPixels` handles letterboxing and black borders

**Implementation** in `VDF.Core/Utils/GrayBytesUtils.cs`:
```csharp
// Pixels outside threshold are skipped in difference calculation
if (ignoreBlackPixels && (img1[i] < 0x20 || img2[i] < 0x20)) continue;
if (ignoreWhitePixels && (img1[i] > 0xF0 || img2[i] > 0xF0)) continue;
```

---

## 4. Running on Linux

**YES - Fully supported!** The project is cross-platform by design.

### Technology Stack

| Component | Technology | Platform Support |
|-----------|------------|------------------|
| Runtime | .NET 9.0 | Windows, Linux, macOS |
| GUI Framework | Avalonia | Cross-platform (like Electron for .NET) |
| Video Processing | FFmpeg | Cross-platform |

### Build Targets

From `VDF.GUI/VDF.GUI.csproj`:
```xml
<TargetFramework>net9.0</TargetFramework>
<RuntimeIdentifiers>win-x64;linux-x64;osx-x64;osx-arm64</RuntimeIdentifiers>
```

### Installation on Linux

1. **Install .NET 9 SDK:**
   ```bash
   # Ubuntu/Debian
   wget https://dot.net/v1/dotnet-install.sh
   chmod +x dotnet-install.sh
   ./dotnet-install.sh --channel 9.0
   ```

2. **Install FFmpeg:**
   ```bash
   sudo apt install ffmpeg
   ```

3. **Build and Run:**
   ```bash
   git clone https://github.com/0x90d/videoduplicatefinder.git
   cd videoduplicatefinder
   dotnet build
   dotnet run --project VDF.GUI
   ```

### Linux-Specific Features

- **POSIX support** via `Mono.Posix.NETStandard` for hardlink detection
- **VAAPI hardware acceleration** for Intel/AMD GPUs
- Native file system operations

---

## 5. How .NET-Bound is the Logic? (Python Conversion Analysis)

### Core Algorithm Components - Portability

| Component | .NET Implementation | Python Equivalent | Difficulty |
|-----------|---------------------|-------------------|------------|
| FFmpeg frame extraction | `FFmpeg.AutoGen` / subprocess | `ffmpeg-python` / subprocess | **Easy** |
| Grayscale conversion | `SixLabors.ImageSharp` | `Pillow` or `OpenCV` | **Easy** |
| DCT perceptual hash | Custom in `PerceptualHash.cs` | `imagehash` library | **Trivial** |
| Pixel comparison | SIMD (AVX2/SSE2) | `numpy` vectorized | **Easy** |
| Database storage | Protocol Buffers | `protobuf` package | **Easy** |
| Parallel processing | `Parallel.ForEach` | `multiprocessing` | **Easy** |

### The Core pHash Algorithm

The DCT-based perceptual hash is standard math, easily portable:

```python
# Python equivalent of VDF.Core/pHash/PerceptualHash.cs
import numpy as np
from scipy.fftpack import dct

def compute_phash(gray_32x32: np.ndarray) -> int:
    """
    Compute perceptual hash from 32x32 grayscale image.
    Returns 64-bit hash as integer.
    """
    # Convert to float
    pixels = gray_32x32.astype(np.float64)

    # 2D DCT (Discrete Cosine Transform)
    dct_result = dct(dct(pixels.T, norm='ortho').T, norm='ortho')

    # Extract 8x8 low-frequency coefficients (excluding DC component)
    low_freq = dct_result[1:9, 1:9].flatten()

    # Binary hash: 1 if above median, 0 if below
    median = np.median(low_freq)
    bits = ''.join(['1' if x > median else '0' for x in low_freq])

    return int(bits, 2)

def hamming_distance(hash1: int, hash2: int) -> int:
    """Compare two hashes. Returns 0-64 (0 = identical)."""
    return bin(hash1 ^ hash2).count('1')

def similarity(hash1: int, hash2: int) -> float:
    """Returns 0.0 to 1.0 similarity score."""
    return 1.0 - (hamming_distance(hash1, hash2) / 64.0)
```

### Grayscale Comparison (Vectorized)

```python
import numpy as np

def grayscale_difference(img1: np.ndarray, img2: np.ndarray) -> float:
    """
    Compare two 32x32 grayscale images.
    Returns 0.0 (identical) to 1.0 (completely different).
    """
    diff = np.abs(img1.astype(np.int16) - img2.astype(np.int16))
    return np.sum(diff) / (img1.size * 255)

def grayscale_difference_filtered(
    img1: np.ndarray,
    img2: np.ndarray,
    ignore_black: bool = False,
    ignore_white: bool = False
) -> float:
    """Compare with pixel filtering for watermarks."""
    mask = np.ones(img1.shape, dtype=bool)

    if ignore_black:
        mask &= (img1 >= 0x20) & (img2 >= 0x20)
    if ignore_white:
        mask &= (img1 <= 0xF0) & (img2 <= 0xF0)

    if not np.any(mask):
        return 1.0  # No valid pixels to compare

    diff = np.abs(img1[mask].astype(np.int16) - img2[mask].astype(np.int16))
    return np.sum(diff) / (np.sum(mask) * 255)
```

### What's Actually .NET-Specific

1. **SIMD Intrinsics** (`System.Runtime.Intrinsics.X86`)
   - Python alternative: NumPy's vectorized operations (automatic SIMD under the hood)

2. **Parallel Processing** (`Parallel.ForEachAsync`)
   - Python alternative: `concurrent.futures.ProcessPoolExecutor` or `multiprocessing.Pool`

3. **Avalonia GUI**
   - Python alternative: PyQt, Tkinter, or just build a CLI tool

4. **Protocol Buffers**
   - Python has `protobuf` package (same format, compatible)

### Existing Python Libraries

These libraries already implement similar functionality:

| Library | Purpose | Notes |
|---------|---------|-------|
| `imagehash` | pHash, dHash, aHash | Drop-in replacement for hash computation |
| `videohash` | Video fingerprinting | Complete video hashing solution |
| `ffmpeg-python` | FFmpeg bindings | Frame extraction |
| `Pillow` | Image processing | Grayscale conversion, resizing |
| `opencv-python` | Computer vision | Alternative to Pillow |

### Minimal Python Implementation

A basic video duplicate finder in Python:

```python
import subprocess
import numpy as np
from PIL import Image
import imagehash
from io import BytesIO

def extract_frame(video_path: str, position: float) -> Image.Image:
    """Extract frame at position (0.0 to 1.0) from video."""
    # Get duration
    result = subprocess.run([
        'ffprobe', '-v', 'error', '-show_entries', 'format=duration',
        '-of', 'csv=p=0', video_path
    ], capture_output=True, text=True)
    duration = float(result.stdout.strip())

    # Extract frame
    timestamp = duration * position
    result = subprocess.run([
        'ffmpeg', '-ss', str(timestamp), '-i', video_path,
        '-vframes', '1', '-f', 'image2pipe', '-vcodec', 'png', '-'
    ], capture_output=True)

    return Image.open(BytesIO(result.stdout))

def compute_video_hash(video_path: str) -> imagehash.ImageHash:
    """Compute perceptual hash for video."""
    frame = extract_frame(video_path, 0.5)  # Middle frame
    frame = frame.convert('L').resize((32, 32))  # Grayscale 32x32
    return imagehash.phash(frame)

def find_duplicates(video_paths: list, threshold: int = 4) -> list:
    """Find duplicate videos. threshold is max Hamming distance."""
    hashes = {path: compute_video_hash(path) for path in video_paths}
    duplicates = []

    paths = list(hashes.keys())
    for i, path1 in enumerate(paths):
        for path2 in paths[i+1:]:
            distance = hashes[path1] - hashes[path2]
            if distance <= threshold:
                duplicates.append((path1, path2, distance))

    return duplicates
```

### Conversion Effort Estimate

| Component | Lines of Code | Python Effort |
|-----------|---------------|---------------|
| pHash algorithm | ~100 | Use `imagehash` library |
| Grayscale comparison | ~150 | ~30 lines with NumPy |
| FFmpeg integration | ~500 | ~100 lines with subprocess |
| Database management | ~200 | ~100 lines with SQLite |
| Scan orchestration | ~400 | ~200 lines |
| **Total core logic** | ~1350 | ~500 lines |

**Verdict**: Converting to Python is **straightforward**. The algorithms are portable math, Python has equivalent libraries, and NumPy provides automatic vectorization.

---

## Summary Table

| Question | Answer |
|----------|--------|
| Scan drives separately? | **YES** - use `IncludeNonExistingFiles` + `ScanAgainstEntireDatabase` |
| Parameters scan vs compare? | **Both** - frame count at scan, threshold/duration at comparison |
| Different durations? | `PercentDurationDifference = 20%` allows ±20% variance |
| Watermarks? | `IgnoreWhitePixels` / `IgnoreBlackPixels` skips those pixels |
| Linux support? | **YES** - .NET 9 + Avalonia, fully cross-platform |
| Python port difficulty? | **Low** - core is portable math, libraries exist |
