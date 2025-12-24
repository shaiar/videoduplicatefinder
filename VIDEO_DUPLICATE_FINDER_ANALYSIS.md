# Video Duplicate Finder - Technical Analysis

## Overview

Video Duplicate Finder is a .NET 9 cross-platform application for detecting duplicate and similar videos. It uses perceptual hashing and grayscale pixel comparison algorithms to find duplicates even when videos have been re-encoded, resized, or slightly modified.

---

## How It Works

### Detection Pipeline

```
1. File Discovery → 2. Frame Extraction → 3. Fingerprint Generation → 4. Comparison → 5. Grouping
```

#### Stage 1: File Discovery
- Recursively scans directories for video/image files
- Filters by blacklist, file size, and path patterns
- Extracts metadata using FFprobe (duration, resolution, codecs, bitrate)

#### Stage 2: Frame Extraction
- Extracts frames at calculated positions across video duration
- Default: 1 frame at 50% of video duration (configurable)
- Each frame is resized to **32×32 pixels grayscale**
- Uses FFmpeg with hardware acceleration support (CUDA, VAAPI, DXVA2, etc.)

#### Stage 3: Fingerprint Generation
Two fingerprinting methods are generated:

1. **Perceptual Hash (pHash)**: 64-bit compact fingerprint using DCT
2. **Grayscale Bytes**: 1,024 bytes (32×32 pixels) raw pixel data

#### Stage 4: Comparison
- Compares each file against all others
- Uses threshold-based matching (default 96% similarity)
- Supports horizontal flip detection
- Filters by duration variance (±20% tolerance)

#### Stage 5: Result Grouping
- Groups duplicates using transitive closure (if A=B and B=C, then A=B=C)
- Ranks files by quality metrics (resolution, bitrate, FPS, file size)

---

## Video Analysis Technology

### Frame Extraction (FFmpeg)

The tool uses FFmpeg in two modes:

**Native Binding Mode** (fastest):
```
Uses FFmpeg.AutoGen for direct P/Invoke to FFmpeg libraries
Supports hardware acceleration
```

**Process Mode** (fallback):
```bash
ffmpeg -ss <position> -i <file> -vf "scale=32:32,format=gray" \
       -f rawvideo -pix_fmt gray -frames:v 1 pipe:1
```

### Image Processing

Uses **SixLabors.ImageSharp** for:
- Bicubic scaling to 32×32
- Grayscale conversion (L8 format - 8-bit luminance)
- Image flipping for mirror detection

---

## Unique Fingerprint System

### YES - The tool creates indexable fingerprints!

The system generates **two types of fingerprints** per video:

### 1. Perceptual Hash (pHash) - Compact Fingerprint

**Location**: `VDF.Core/pHash/PerceptualHash.cs`

**Algorithm**: Discrete Cosine Transform (DCT)-based perceptual hashing

```
Input: 32×32 grayscale image (1,024 bytes)
    ↓
1. Convert to float array
2. Apply 2D Discrete Cosine Transform (DCT)
   - Row-wise DCT (32×32)
   - Column-wise DCT (32×32)
3. Extract 8×8 AC coefficients (top-left, excluding DC)
4. Calculate median of 64 values
5. Binary threshold (1 if > median, 0 if ≤)
    ↓
Output: 64-bit unsigned integer (ulong)
```

**Key Properties**:
| Property | Value |
|----------|-------|
| Size | 64 bits (8 bytes) |
| Comparison | Hamming distance |
| Similarity | `1 - (hamming_distance / 64)` |
| Indexable | YES - can be stored and compared without original file |

**Comparison Code**:
```csharp
static int Hamming(ulong a, ulong b) => BitOperations.PopCount(a ^ b);
// 0 = identical, 64 = completely different
```

### 2. Grayscale Bytes - Detailed Fingerprint

**Size**: 1,024 bytes per frame position

**Comparison**: Absolute difference sum with SIMD optimization (AVX2/SSE2)

```csharp
for each pixel: diff += |img1[i] - img2[i]|
similarity = 1 - (diff / (imageSize * 255))
```

**Special Features**:
- Can ignore black pixels (< 0x20) - handles letterboxing
- Can ignore white pixels (> 0xF0) - handles burned-in subtitles

---

## Database / Index Storage

The fingerprints ARE stored persistently in a database!

**Location**: `ScannedFiles.db` (configurable)

**Format**: Protocol Buffers (protobuf-net)

**Stored Per File**:
```csharp
class FileEntry {
    string Path;
    DateTime DateCreated, DateModified;
    long FileSize;
    MediaInfo mediaInfo;                    // Duration, resolution, codecs
    Dictionary<double, byte[]> grayBytes;   // Grayscale per position
    Dictionary<double, ulong?> PHashes;     // pHash per position
}
```

**Benefits**:
- Rescans only process new/modified files
- Fingerprints persist across sessions
- Can compare against indexed files without accessing original video

---

## Can the Core Logic Be Extracted?

### YES - The architecture already supports this!

The codebase is cleanly separated:

```
VideoDuplicateFinder/
├── VDF.Core/           ← CORE LIBRARY (extractable)
│   ├── FFTools/        ← FFmpeg integration
│   ├── pHash/          ← Perceptual hashing
│   ├── Utils/          ← Comparison utilities
│   ├── ScanEngine.cs   ← Main orchestration
│   └── DatabaseUtils.cs ← Index persistence
│
└── VDF.GUI/            ← UI (not needed for CLI tool)
```

### Core Components to Extract

| Component | File | Purpose |
|-----------|------|---------|
| Fingerprint Generation | `PerceptualHash.cs` | DCT-based 64-bit hash |
| Frame Extraction | `FfmpegEngine.cs` | Extract grayscale frames |
| Comparison | `GrayBytesUtils.cs`, `PHashCompare.cs` | SIMD-optimized comparison |
| Database | `DatabaseUtils.cs` | Persistent fingerprint storage |
| Scan Engine | `ScanEngine.cs` | Orchestration logic |

### Minimal CLI Tool Requirements

To create a standalone fingerprinting/comparison tool, you need:

1. **FFmpeg binaries** (ffmpeg, ffprobe)
2. **VDF.Core** project (or extract key classes)
3. **Dependencies**:
   - `FFmpeg.AutoGen` - Native FFmpeg bindings
   - `SixLabors.ImageSharp` - Image processing
   - `protobuf-net` - Database serialization (optional)

### Proposed CLI Tool API

```csharp
// Generate fingerprint for a video
var fingerprint = VideoFingerprint.Generate("video.mp4");
// Returns: { pHash: 0x1234567890ABCDEF, grayBytes: byte[1024] }

// Compare two videos
float similarity = VideoFingerprint.Compare(fp1, fp2);
// Returns: 0.0 to 1.0

// Index a directory
var index = new VideoIndex("index.db");
index.AddDirectory("/videos");

// Find duplicates
var duplicates = index.FindDuplicates(threshold: 0.96f);
```

---

## Key Configuration Options

| Setting | Default | Description |
|---------|---------|-------------|
| `Percent` | 96% | Similarity threshold |
| `ThumbnailCount` | 1 | Frames sampled per video |
| `PercentDurationDifference` | 20% | Duration variance tolerance |
| `UsePHashing` | false | Use pHash instead of graybytes |
| `CompareHorizontallyFlipped` | false | Detect mirrored videos |
| `IgnoreBlackPixels` | false | Ignore letterboxing |
| `IgnoreWhitePixels` | false | Ignore burned subtitles |

---

## Performance Optimizations

1. **SIMD Vectorization**: AVX2/SSE2 for pixel comparisons
2. **Hardware Acceleration**: FFmpeg with GPU decoding
3. **Parallel Processing**: `Parallel.ForEachAsync` for file processing
4. **Array Pooling**: Reused buffers to minimize GC
5. **Early Exit**: Stops comparison when threshold exceeded
6. **Persistent Cache**: Avoids re-extracting frames on rescans

---

## Limitations & Considerations

1. **Single Frame Default**: May miss duplicates with different intros
   - Solution: Increase `ThumbnailCount` to 3-5

2. **No Audio Comparison**: Only visual fingerprinting
   - Could be extended with audio fingerprinting (Chromaprint)

3. **No Scene Detection**: Fixed position sampling
   - Could be improved with content-aware keyframe extraction

4. **Database Format**: Protobuf is binary, not queryable
   - Consider SQLite for a production indexing system

---

## Summary

| Question | Answer |
|----------|--------|
| How does it work? | Frame extraction → Grayscale resize → pHash/byte fingerprint → Threshold comparison |
| What analyzes videos? | FFmpeg for frame extraction, DCT for hashing, SIMD for comparison |
| Unique fingerprint? | **YES** - 64-bit pHash or 1KB grayscale per video, stored in database |
| Indexable? | **YES** - Fingerprints persist in ScannedFiles.db, can compare without original files |
| Extractable core? | **YES** - VDF.Core is already a separate library, can be wrapped as CLI tool |

---

## Next Steps for a Dedicated Tool

1. **Extract VDF.Core** into standalone NuGet package
2. **Create CLI wrapper** with commands: `index`, `compare`, `find-duplicates`
3. **Replace Protobuf database** with SQLite for queryability
4. **Add REST API** for integration with other systems
5. **Consider audio fingerprinting** (Chromaprint/AcoustID) for complete media matching
