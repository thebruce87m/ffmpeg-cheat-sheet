# ffmpeg-cheat-sheet

# Record from a hikvision NVR in chunks:

```bash
ffmpeg -y \
-rtsp_transport tcp \
-i rtsp://username:password@192.168.1.30:554/Streaming/tracks/201?starttime=20231003T075500Z \
-ss 00:00:0 \
-t 00:01:00 \
-acodec copy \
-vcodec copy \
-f segment \
-segment_time 10 \
./output/xxx-%03d.mkv
```

Will record in chunks as follows:

```bash
xxx-000.mkv
xxx-001.mkv
xxx-002.mkv
xxx-003.mkv
xxx-004.mkv
```


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

# A script to generate a mosaic script

```bash
#!/bin/bash

# Desired output resolution
OUTPUT_RES_X=3840
OUTPUT_RES_Y=2160

# Number of videos in each row
OUTPUT_STACK_X=2

# Number of videos in each column
OUTPUT_STACK_Y=2

# Calculate each sub-videos (tile) size
TILE_X_RES=$(($OUTPUT_RES_X / $OUTPUT_STACK_X))
TILE_Y_RES=$(($OUTPUT_RES_Y / $OUTPUT_STACK_Y))

# Find the files for our mosaic
FILES=($(find recordings2/ -name "*.mp4" | sort))

echo "Found ${#FILES[@]} Files"


#
# Header
#
cat <<EOF >run_mosaic.sh
#!/bin/bash

ffmpeg \\
EOF

#
# Include the files
#
for FILE in "${FILES[@]}"
do
echo $FILE

cat <<EOF >>run_mosaic.sh
-i $FILE \\
EOF
done

#
#
#
cat <<EOF >>run_mosaic.sh
\\
-filter_complex " \\
\\
nullsrc=size=${OUTPUT_RES_X}x${OUTPUT_RES_Y} [base]; \\
\\
EOF

#
#
#

COUNT=0

for FILE in "${FILES[@]}"
do

cat <<EOF >>run_mosaic.sh
[$COUNT:v] setpts=PTS-STARTPTS, scale=${TILE_X_RES}x${TILE_Y_RES} [V$COUNT]; \\
EOF

COUNT=$(($COUNT + 1))

done

#
# Newline
#
echo -e "\\" >> run_mosaic.sh

#
# Create our overlay lines
#

COUNT=0
FILE_TILE_X_RES=0
FILE_TILE_Y_RES=0

for FILE in "${FILES[@]}"
do

# Work out our temp frame names
if [ "$COUNT" -eq "0" ]; then
  LEFT="base"
  RIGHT="tmp$(($COUNT + 1))"
else
  LEFT="tmp$COUNT"
  RIGHT="tmp$(($COUNT + 1))"
fi

#The last entry doesn't create a temp frame

if [ "$FILE" = "${FILES[-1]}" ]; then

cat <<EOF >>run_mosaic.sh
[$LEFT][V$COUNT] overlay=shortest=1:x=$FILE_TILE_X_RES:y=$FILE_TILE_Y_RES \\
EOF

else

cat <<EOF >>run_mosaic.sh
[$LEFT][V$COUNT] overlay=shortest=1:x=$FILE_TILE_X_RES:y=$FILE_TILE_Y_RES [$RIGHT]; \\
EOF

fi



COUNT=$(($COUNT + 1))

# Calculate the X position for the next tile
FILE_TILE_X_RES=$(($FILE_TILE_X_RES + $TILE_X_RES))

# If the tile will overrun the max X resolution
# then wrap the video to the next line
if [ "$(($FILE_TILE_X_RES + $TILE_X_RES))" -gt "$OUTPUT_RES_X" ]; then
FILE_TILE_X_RES=0
FILE_TILE_Y_RES=$(($FILE_TILE_Y_RES + $TILE_Y_RES))
fi


done


#
#
#
cat <<EOF >>run_mosaic.sh
" \\
-c:v libx264 output.mkv
EOF


chmod u+x run_mosaic.sh
```

example output:

```bash
#!/bin/bash

ffmpeg \
-i recordings2/192.168.250.100/file-000.mp4 \
-i recordings2/192.168.250.101/file-000.mp4 \
-i recordings2/192.168.250.102/file-000.mp4 \
-i recordings2/192.168.250.103/file-000.mp4 \
\
-filter_complex " \
\
nullsrc=size=3840x2160 [base]; \
\
[0:v] setpts=PTS-STARTPTS, scale=1920x1080 [V0]; \
[1:v] setpts=PTS-STARTPTS, scale=1920x1080 [V1]; \
[2:v] setpts=PTS-STARTPTS, scale=1920x1080 [V2]; \
[3:v] setpts=PTS-STARTPTS, scale=1920x1080 [V3]; \
\
[base][V0] overlay=shortest=1:x=0:y=0 [tmp1]; \
[tmp1][V1] overlay=shortest=1:x=1920:y=0 [tmp2]; \
[tmp2][V2] overlay=shortest=1:x=0:y=1080 [tmp3]; \
[tmp3][V3] overlay=shortest=1:x=1920:y=1080 \
" \
-c:v libx264 output.mkv

```
