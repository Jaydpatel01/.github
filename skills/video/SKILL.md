---
name: video
description: Read, analyze, extract information from, and process video files. Use this skill whenever the user wants to work with video files (.mp4, .mov, .avi, .mkv, .webm, .flv, .wmv, etc.). This includes extracting frames or thumbnails, analyzing video content, extracting audio, getting video metadata (duration, resolution, codec, frame rate), transcribing speech from video, detecting scenes or key moments, generating summaries or descriptions of video content, converting between video formats, and trimming or clipping videos. Trigger whenever the user mentions a video file or asks to process, analyze, or extract anything from a video.
---

# Video Processing Guide

## Overview

This skill provides tools and workflows for reading and analyzing video files. Common tasks include extracting frames, analyzing content, extracting audio, transcribing speech, and getting metadata.

## Quick Reference

| Task | Tool | Command/Library |
|------|------|-----------------|
| Metadata | ffprobe | `ffprobe -v quiet -print_format json -show_format -show_streams video.mp4` |
| Extract frames | ffmpeg | `ffmpeg -i video.mp4 -vf fps=1 frame_%04d.png` |
| Extract audio | ffmpeg | `ffmpeg -i video.mp4 -q:a 0 -map a audio.mp3` |
| Thumbnails | ffmpeg | `ffmpeg -i video.mp4 -ss 00:00:05 -vframes 1 thumbnail.png` |
| Transcription | whisper | `whisper audio.mp3 --model base` |
| Python video I/O | opencv-python | `cv2.VideoCapture("video.mp4")` |
| Scene detection | scenedetect | `scenedetect -i video.mp4 detect-content` |

---

## Reading Video Metadata

### Using ffprobe (recommended)

```bash
# Full metadata as JSON
ffprobe -v quiet -print_format json -show_format -show_streams video.mp4

# Quick summary
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,duration,codec_name \
  -of default=noprint_wrappers=1 video.mp4
```

### Using Python (opencv)

```python
import cv2

cap = cv2.VideoCapture("video.mp4")

# Basic properties
fps = cap.get(cv2.CAP_PROP_FPS)
frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
duration_seconds = frame_count / fps

print(f"Resolution: {width}x{height}")
print(f"FPS: {fps}")
print(f"Total frames: {frame_count}")
print(f"Duration: {duration_seconds:.2f}s")

cap.release()
```

### Using Python (moviepy)

```python
from moviepy.editor import VideoFileClip

clip = VideoFileClip("video.mp4")
print(f"Duration: {clip.duration}s")
print(f"FPS: {clip.fps}")
print(f"Size: {clip.size}")
clip.close()
```

---

## Extracting Frames

### Extract frames at regular intervals

```bash
# Extract 1 frame per second
ffmpeg -i video.mp4 -vf fps=1 frames/frame_%04d.png

# Extract every 5 seconds
ffmpeg -i video.mp4 -vf fps=1/5 frames/frame_%04d.png

# Extract specific time range (from 10s to 20s, 2fps)
ffmpeg -ss 10 -to 20 -i video.mp4 -vf fps=2 frames/frame_%04d.png
```

### Extract frames with Python (opencv)

```python
import cv2
import os

def extract_frames(video_path, output_dir, interval_seconds=1):
    """Extract frames at specified intervals."""
    os.makedirs(output_dir, exist_ok=True)
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    frame_interval = int(fps * interval_seconds)
    
    frame_idx = 0
    saved_count = 0
    
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        if frame_idx % frame_interval == 0:
            output_path = os.path.join(output_dir, f"frame_{saved_count:04d}.jpg")
            cv2.imwrite(output_path, frame)
            saved_count += 1
        frame_idx += 1
    
    cap.release()
    print(f"Extracted {saved_count} frames to {output_dir}")
    return saved_count

extract_frames("video.mp4", "frames/", interval_seconds=2)
```

### Extract thumbnails at key timestamps

```bash
# Extract thumbnail at 5 seconds
ffmpeg -i video.mp4 -ss 00:00:05 -vframes 1 thumbnail.png

# Extract multiple thumbnails
ffmpeg -i video.mp4 -vf "thumbnail,scale=320:-1" -frames:v 1 thumb.png
```

---

## Extracting Audio

```bash
# Extract audio as MP3
ffmpeg -i video.mp4 -q:a 0 -map a audio.mp3

# Extract audio as WAV (better for transcription)
ffmpeg -i video.mp4 -acodec pcm_s16le -ar 16000 audio.wav

# Extract audio from specific time range
ffmpeg -i video.mp4 -ss 00:00:30 -to 00:01:00 -q:a 0 -map a clip_audio.mp3
```

---

## Transcribing Speech

### Using OpenAI Whisper

```bash
# Install
pip install openai-whisper

# Transcribe (auto-detect language)
whisper audio.mp3 --model base

# Transcribe with specific language and output format
whisper audio.mp3 --model medium --language English --output_format srt
```

```python
import whisper

model = whisper.load_model("base")  # Options: tiny, base, small, medium, large

# Transcribe from audio file
result = model.transcribe("audio.mp3")
print(result["text"])

# Transcribe directly from video (whisper handles audio extraction)
result = model.transcribe("video.mp4")
print(result["text"])

# Get timestamped segments
for segment in result["segments"]:
    start = segment["start"]
    end = segment["end"]
    text = segment["text"]
    print(f"[{start:.1f}s - {end:.1f}s] {text}")
```

### Full workflow: video → transcription

```python
import subprocess
import whisper
import os

def transcribe_video(video_path, model_size="base"):
    """Extract audio from video and transcribe it."""
    # Extract audio
    audio_path = video_path.rsplit(".", 1)[0] + "_audio.wav"
    subprocess.run([
        "ffmpeg", "-i", video_path,
        "-acodec", "pcm_s16le", "-ar", "16000",
        "-y", audio_path
    ], check=True, capture_output=True)
    
    # Transcribe
    model = whisper.load_model(model_size)
    result = model.transcribe(audio_path)
    
    # Cleanup
    os.remove(audio_path)
    
    return result

result = transcribe_video("video.mp4")
print(result["text"])
```

---

## Analyzing Video Content

### Scene Detection

```bash
# Install PySceneDetect
pip install scenedetect[opencv]

# Detect scene cuts
scenedetect -i video.mp4 detect-content list-scenes

# Save thumbnails at each scene change
scenedetect -i video.mp4 detect-content save-images
```

```python
from scenedetect import VideoManager, SceneManager
from scenedetect.detectors import ContentDetector

def detect_scenes(video_path, threshold=30.0):
    """Detect scene changes in a video."""
    video_manager = VideoManager([video_path])
    scene_manager = SceneManager()
    scene_manager.add_detector(ContentDetector(threshold=threshold))
    
    video_manager.start()
    scene_manager.detect_scenes(frame_source=video_manager)
    scene_list = scene_manager.get_scene_list()
    video_manager.release()
    
    print(f"Detected {len(scene_list)} scenes:")
    for i, (start, end) in enumerate(scene_list):
        print(f"  Scene {i+1}: {start.get_timecode()} → {end.get_timecode()}")
    
    return scene_list

detect_scenes("video.mp4")
```

### Analyzing frames with vision models

```python
import cv2
import base64
from pathlib import Path

def extract_key_frames(video_path, num_frames=5):
    """Extract evenly spaced frames for visual analysis."""
    cap = cv2.VideoCapture(video_path)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    frames = []
    for i in range(num_frames):
        frame_pos = int(total_frames * i / num_frames)
        cap.set(cv2.CAP_PROP_POS_FRAMES, frame_pos)
        ret, frame = cap.read()
        if ret:
            # Convert to JPEG bytes
            _, buffer = cv2.imencode('.jpg', frame)
            frame_b64 = base64.b64encode(buffer).decode('utf-8')
            frames.append({
                "frame_number": frame_pos,
                "timestamp": frame_pos / cap.get(cv2.CAP_PROP_FPS),
                "data": frame_b64
            })
    
    cap.release()
    return frames

# Use with a vision model to describe video content
frames = extract_key_frames("video.mp4", num_frames=5)
print(f"Extracted {len(frames)} key frames for analysis")
```

---

## Video Conversion and Clipping

### Convert between formats

```bash
# Convert to MP4 (H.264)
ffmpeg -i input.avi -c:v libx264 -c:a aac output.mp4

# Convert to WebM (VP9)
ffmpeg -i input.mp4 -c:v libvpx-vp9 -c:a libopus output.webm

# Reduce file size
ffmpeg -i input.mp4 -crf 28 -preset fast output_compressed.mp4
```

### Trim/clip video

```bash
# Trim from 30s to 1m30s
ffmpeg -ss 00:00:30 -to 00:01:30 -i input.mp4 -c copy clip.mp4

# First 60 seconds
ffmpeg -i input.mp4 -t 60 -c copy first_minute.mp4
```

### Python clipping with moviepy

```python
from moviepy.editor import VideoFileClip

clip = VideoFileClip("video.mp4")

# Subclip from 10s to 25s
subclip = clip.subclip(10, 25)
subclip.write_videofile("clip.mp4")

clip.close()
```

---

## Working with Video Streams

### Read video frame by frame (for processing)

```python
import cv2

def process_video(video_path):
    """Process each frame in a video."""
    cap = cv2.VideoCapture(video_path)
    
    fps = cap.get(cv2.CAP_PROP_FPS)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    frame_idx = 0
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        
        # Process frame here (e.g., detect objects, analyze content)
        timestamp = frame_idx / fps
        
        # Example: print progress every 100 frames
        if frame_idx % 100 == 0:
            progress = frame_idx / total_frames * 100
            print(f"Processing frame {frame_idx}/{total_frames} ({progress:.1f}%) at {timestamp:.2f}s")
        
        frame_idx += 1
    
    cap.release()
    print(f"Processed {frame_idx} frames")
```

---

## Supported Video Formats

| Format | Extension | Notes |
|--------|-----------|-------|
| MPEG-4 | .mp4 | Most common, widely supported |
| QuickTime | .mov | Common on macOS/iOS |
| AVI | .avi | Older Windows format |
| Matroska | .mkv | Container for many codecs |
| WebM | .webm | Web-optimized format |
| Flash Video | .flv | Legacy web format |
| Windows Media | .wmv | Windows-native format |
| MPEG-2 | .mpg, .mpeg | Broadcast/DVD format |
| 3GPP | .3gp | Mobile video format |

---

## Dependencies

Install required tools and libraries:

```bash
# System tools (install via package manager)
# Ubuntu/Debian: sudo apt install ffmpeg
# macOS: brew install ffmpeg

# Python libraries
pip install opencv-python          # Video reading/writing
pip install moviepy                # High-level video editing
pip install openai-whisper         # Speech transcription
pip install scenedetect[opencv]    # Scene detection
pip install imageio[ffmpeg]        # Video I/O via imageio
```

### Checking ffmpeg installation

```bash
ffmpeg -version
ffprobe -version
```

---

## Best Practices

1. **Check file integrity first**: Use `ffprobe` to verify the video is readable before processing
2. **Use ffmpeg for format conversion**: It handles codec details automatically
3. **Extract audio before transcribing**: Whisper works best on clean audio; isolate it from video first
4. **Sample frames for analysis**: Don't analyze every frame — use intervals or scene detection
5. **Handle large files**: Process in chunks or use streaming approaches for long videos
6. **Preserve originals**: Always work on copies; never modify the source file in place
7. **Check available memory**: Loading entire videos into memory can crash for large files — use frame-by-frame processing with opencv instead
