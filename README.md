# ffmpeg-cheat-sheet


# Make a mosaic from 4 videos

Creates a video called `output.mkv` with 640 x 480 resolution

```bash
ffmpeg \
\
-i file1.mp4 \
-i file2.mp4 \
-i file3.mp4 \
-i file4.mp4 \
\
-filter_complex " \
\
nullsrc=size=640x480 [base]; \
\
[0:v] setpts=PTS-STARTPTS, scale=320x240 [upperleft]; \
[1:v] setpts=PTS-STARTPTS, scale=320x240 [upperright]; \
[2:v] setpts=PTS-STARTPTS, scale=320x240 [lowerleft]; \
[3:v] setpts=PTS-STARTPTS, scale=320x240 [lowerright]; \
\
[base][upperleft]  overlay=shortest=1:x=000:y=000 [tmp1]; \
[tmp1][upperright] overlay=shortest=1:x=320:y=000 [tmp2]; \
[tmp2][lowerleft]  overlay=shortest=1:x=000:y=240 [tmp3]; \
[tmp3][lowerright] overlay=shortest=1:x=320:y=240 \
" \
-c:v libx264 output.mkv
```
