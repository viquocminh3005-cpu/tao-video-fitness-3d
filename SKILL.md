$skill = Join-Path (Get-Location) '.agents\skills\tao-video-fitness-3d'
if (!(Test-Path -LiteralPath (Join-Path $skill 'SKILL.md'))) {
    throw 'Hay mo PowerShell tai thu muc goc cua repo da chua skill tu bo truoc.'
}

$scripts = Join-Path $skill 'scripts'
New-Item -ItemType Directory -Force -Path $scripts | Out-Null
$utf8 = [System.Text.UTF8Encoding]::new($false)
[System.IO.File]::WriteAllText((Join-Path $skill 'SKILL.md'), @'
---

name: tao-video-fitness-3d
description: >-
  Biến kịch bản bài tập thành video fitness 3D dọc có phụ đề và xuất MP4.
  Dùng cho video hướng dẫn tập luyện trên TikTok, Facebook Reels và YouTube Shorts.

---

# Tạo video fitness 3D từ kịch bản

1. Đọc toàn bộ kịch bản và giữ nguyên thứ tự bài, số hiệp, số lần lặp và
   lời thoại do người dùng cung cấp. Nếu chưa có thời lượng, ước lượng từ
   lời thoại và số cảnh. Hỏi khi tên hoặc mô tả bài mơ hồ đến mức có thể
   dựng nhầm động tác.
2. Đọc [hướng dẫn sản xuất](references/quy-trinh-san-xuat.md). Giữ nhất
   quán nhân vật, trang phục và ánh sáng. Với mỗi bài, thể hiện tư thế đầu
   và cuối, chọn góc thấy rõ các khớp cần theo dõi. Dùng thiết kế riêng
   cho nhân vật và bối cảnh.
3. Tạo tư liệu từng cảnh trong một thư mục làm việc. Ưu tiên clip chuyển
   động nếu có công cụ tạo video và kiểm tra được động tác. Nếu có công
   cụ tạo ảnh, tạo các ảnh tư thế đầu và cuối, dùng từng ảnh làm một cảnh
   theo đúng trình tự. Dùng tư liệu người dùng cung cấp khi phù hợp. Nếu
   không có công cụ tạo hình hoặc tư liệu đầu vào, báo thiếu tư liệu cụ thể.
4. Tạo `project.json` theo [mẫu](references/project.example.json). Mỗi
   cảnh có `media`, `duration` và có thể có `caption`. Dùng đường dẫn
   tương đối theo vị trí JSON. Tách lời thoại thành phụ đề ngắn, tối đa hai
   dòng mỗi cảnh. Khi có công cụ tạo giọng hoặc người dùng gửi bản thu,
   đặt file giọng đọc vào trường `audio`. Canh thứ tự và thời lượng cảnh
   theo bản thu trước khi dựng.
5. Chạy `python "<thư mục skill>/scripts/render.py" "<project.json>"
   "<video.mp4>"`. Máy thực thi cần Python 3.9 trở lên, Pillow, FFmpeg và
   ffprobe trong PATH. Có thể đặt trường `font` trong JSON để chọn font
   TrueType có đủ dấu tiếng Việt.
6. Mở file thành phẩm khi có công cụ kiểm tra hình ảnh. Đối chiếu từng
   bài với kịch bản, xem các tư thế, dấu tiếng Việt và sự đồng bộ với
   giọng đọc. Sửa cảnh sai và dựng lại. Bàn giao đường dẫn MP4 đã xuất.
   Ghi rõ hình thức ghép ảnh hoặc tình trạng thiếu giọng đọc nếu có.
   Khi dựng thất bại, nêu lỗi cụ thể và tình trạng file.
   '@, $utf8)

[System.IO.File]::WriteAllText((Join-Path $skill 'references\project.example.json'), @'
{
  "scenes": [
    {
      "media": "media/01-squat-dung.png",
      "duration": 2,
      "caption": "Squat\nBắt đầu ở tư thế đứng"
    },
    {
      "media": "media/02-squat-ha.png",
      "duration": 3,
      "caption": "Hạ người có kiểm soát"
    },
    {
      "media": "media/03-nhon-got.mp4",
      "duration": 5,
      "caption": "Nhón gót\nNâng gót rồi hạ chậm"
    }
  ]
}
'@, $utf8)

[System.IO.File]::WriteAllText((Join-Path $scripts 'requirements.txt'), "Pillow>=10`n", $utf8)

[System.IO.File]::WriteAllText((Join-Path $scripts 'render.py'), @'
import argparse
import json
import math
import os
from pathlib import Path
import shutil
import subprocess
import sys
import tempfile

from PIL import Image, ImageDraw, ImageFont

WIDTH = 1080
HEIGHT = 1920
FPS = 30
IMAGE_EXTENSIONS = {".png", ".jpg", ".jpeg", ".webp"}
VIDEO_EXTENSIONS = {".mp4", ".mov", ".mkv", ".webm"}

def run(*args):
    result = subprocess.run(
        [str(arg) for arg in args],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        encoding="utf-8",
        errors="replace",
    )
    if result.returncode:
        raise RuntimeError(result.stderr.strip() or f"{args[0]} failed")
    return result.stdout

def asset(base, value):
    if not isinstance(value, str) or not value.strip():
        raise ValueError("Asset paths must be nonempty strings.")
    path = (base / value).resolve()
    if not path.is_file():
        raise ValueError(f"Missing asset: {path}")
    return path

def caption_font(base, custom_font):
    if custom_font is not None:
        return asset(base, custom_font)
    windows_fonts = Path(os.environ.get("WINDIR", "C:/Windows")) / "Fonts"
    for candidate in (
        windows_fonts / "arialbd.ttf",
        Path("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"),
        Path("/System/Library/Fonts/Supplemental/Arial Bold.ttf"),
    ):
        if candidate.is_file():
            return candidate
    raise ValueError("Set 'font' in project.json to a Vietnamese-capable TTF.")

def make_overlay(path, caption, font_path):
    overlay = Image.new("RGBA", (WIDTH, HEIGHT))
    if not caption.strip():
        overlay.save(path)
        return
    lines = [line.strip() for line in caption.splitlines() if line.strip()]
    if len(lines) > 2:
        raise ValueError(f"Use at most two caption lines: {caption!r}")

    draw = ImageDraw.Draw(overlay)
    for size in range(62, 35, -2):
        font = ImageFont.truetype(str(font_path), size)
        if all(draw.textlength(line, font=font) <= 860 for line in lines):
            break
    else:
        raise ValueError(f"Caption is too wide. Split it across scenes: {caption!r}")

    line_height = size + 20
    bottom = HEIGHT - 300
    top = bottom - 42 - line_height * len(lines)
    draw.rounded_rectangle((68, top, WIDTH - 68, bottom), 24, fill=(0, 0, 0, 205))
    for index, line in enumerate(lines):
        y = top + 21 + index * line_height + line_height / 2
        draw.text((WIDTH // 2, y), line, font=font, fill="white", anchor="mm")
    overlay.save(path)

def render(project_path, output):
    if output.suffix.lower() != ".mp4":
        raise ValueError("Output filename must end in .mp4.")
    for tool in ("ffmpeg", "ffprobe"):
        if shutil.which(tool) is None:
            raise RuntimeError(f"{tool} is missing from PATH.")
    data = json.loads(project_path.read_text(encoding="utf-8-sig"))
    if not isinstance(data, dict) or not isinstance(data.get("scenes"), list):
        raise ValueError("project.json must contain a scenes array.")
    if not data["scenes"]:
        raise ValueError("Add at least one scene.")

    base = project_path.parent
    scenes = []
    for number, scene in enumerate(data["scenes"], 1):
        if not isinstance(scene, dict):
            raise ValueError(f"Scene {number} must be an object.")
        media = asset(base, scene.get("media"))
        duration = scene.get("duration")
        caption = scene.get("caption", "")
        if media.suffix.lower() not in IMAGE_EXTENSIONS | VIDEO_EXTENSIONS:
            raise ValueError(f"Scene {number} uses an unsupported file type: {media}")
        if (
            type(duration) not in (int, float)
            or not math.isfinite(duration)
            or duration < 1 / FPS
        ):
            raise ValueError(f"Scene {number} needs a positive duration of at least one frame.")
        if not isinstance(caption, str):
            raise ValueError(f"Scene {number} caption must be a string.")
        scenes.append((media, round(duration * FPS), caption))

    font = caption_font(base, data.get("font")) if any(
        caption.strip() for _, _, caption in scenes
    ) else None
    audio = asset(base, data["audio"]) if "audio" in data else None
    total = sum(frames for _, frames, _ in scenes) / FPS

    if audio:
        audio_info = json.loads(run(
            "ffprobe", "-v", "error", "-select_streams", "a:0",
            "-show_entries", "stream=index:format=duration", "-of", "json", audio,
        ))
        if not audio_info.get("streams"):
            raise ValueError(f"No audio track in {audio}")
        audio_duration = float(audio_info.get("format", {}).get("duration", "nan"))
        if not math.isfinite(audio_duration):
            raise ValueError("Cannot determine narration duration.")
        if audio_duration > total + 1 / FPS:
            raise ValueError("Narration is longer than the scenes. Increase scene durations.")

    output.parent.mkdir(parents=True, exist_ok=True)
    with tempfile.TemporaryDirectory(prefix="fitness-", dir=output.parent) as directory:
        work = Path(directory)
        manifest = work / "concat.txt"
        entries = []
        for number, (media, frames, caption) in enumerate(scenes, 1):
            overlay = work / f"overlay-{number:03d}.png"
            segment = work / f"scene-{number:03d}.mp4"
            make_overlay(overlay, caption, font)
            source = (
                ["-loop", "1", "-framerate", str(FPS)]
                if media.suffix.lower() in IMAGE_EXTENSIONS
                else ["-stream_loop", "-1"]
            )
            filters = (
                f"[0:v]scale={WIDTH}:{HEIGHT}:force_original_aspect_ratio=decrease:"
                f"force_divisible_by=2,pad={WIDTH}:{HEIGHT}:(ow-iw)/2:(oh-ih)/2,"
                f"setsar=1,fps={FPS}[base];[base][1:v]overlay=format=auto[v]"
            )
            run(
                "ffmpeg", "-v", "error", *source, "-i", media, "-i", overlay,
                "-filter_complex", filters, "-map", "[v]", "-an",
                "-frames:v", frames, "-c:v", "libx264", "-preset", "fast",
                "-crf", "18", "-pix_fmt", "yuv420p", segment,
            )
            entries.append(f"file '{segment.name}'")

        manifest.write_text("\n".join(entries) + "\n", encoding="utf-8")
        final = work / "final.mp4"
        command = ["ffmpeg", "-v", "error", "-f", "concat", "-safe", "1", "-i", manifest]
        if audio:
            command += ["-i", audio, "-map", "0:v:0", "-map", "1:a:0", "-af", "apad", "-c:a", "aac"]
        else:
            command += ["-map", "0:v:0"]
        run(*command, "-t", f"{total:.6f}", "-c:v", "copy", "-movflags", "+faststart", final)

        info = json.loads(run(
            "ffprobe", "-v", "error", "-select_streams", "v:0",
            "-show_entries", "stream=codec_name,width,height:format=duration",
            "-of", "json", final,
        ))
        streams = info.get("streams", [])
        if not streams or (
            streams[0].get("codec_name"),
            streams[0].get("width"),
            streams[0].get("height"),
        ) != ("h264", WIDTH, HEIGHT):
            raise RuntimeError("Output does not contain a 1080 × 1920 H.264 stream.")
        actual_duration = float(info["format"]["duration"])
        if abs(actual_duration - total) > 2 / FPS:
            raise RuntimeError(f"Unexpected output duration: {actual_duration:.2f}s")
        run("ffmpeg", "-v", "error", "-xerror", "-i", final, "-f", "null", "-")
        os.replace(final, output)

    print(f"{output} | {WIDTH}x{HEIGHT} | {actual_duration:.2f}s")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Render a vertical fitness MP4.")
    parser.add_argument("project", type=Path)
    parser.add_argument("output", type=Path)
    args = parser.parse_args()
    try:
        render(args.project.resolve(), args.output.resolve())
    except (ValueError, OSError, RuntimeError) as exc:
        print(f"Render error: {exc}", file=sys.stderr)
        sys.exit(1)
'@, $utf8)

Get-ChildItem -LiteralPath $skill -Recurse -File |
    Select-Object -ExpandProperty FullName
