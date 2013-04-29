---
title : Storyboard
moretitle: Read the TV and Movies you don't want to watch
description: Storyboard Generates Books From Subtitled Video Files
image: storyboard/sample1.jpg
date: '2013-01-20'
versions:
  - name: FFmpeg
    version: '1.1'
  - name: Imagemagick
    version: 6.7.7-6
  - name: Storyboard
    version: 0.3.0

---

[Storyboard](https://github.com/markolson/storyboard) was born of my insane desire to consume videos without actually having to watch them. Normally that would involve putting the TV on in the background and ignoring the video while listening to the audio, but what about the reverse? All visual without the audio. On my kindle.

Storyboard is **very** much a work in progress, though it does work for most files that it has been tested against. Right now it will only create PDFs, but ePUB and Mobi are just a few commits away. The source <a href="https://github.com/markolson/storyboard">is available on Github</a> where pull requests are gladly accepted, especially ones that deal with <a href="https://github.com/markolson/storyboard/issues">open issues.</a> Even just hearing back that Storyboard did or didn't work for a certain file can be of great help.

##Installing
Storyboard requires Ruby 1.9.3, FFmpeg 1.1, and ImageMagick. For OSX, the [INSTALL file](https://github.com/markolson/storyboard/blob/master/INSTALL.md) has a lot more details, but if you already have a development environment setup, you shouldn't have to do much.

    brew update && brew install ffmpeg imagemagick ghostscript libtool
    gem install storyboard

For Ubuntu, installing imagemagick and ghostscript from apt will work, but you need to <a href="http://ffmpeg.org/download.html">download and compile ffmpeg</a> on your own for now.

## Using Storyboard
From a terminal, just call `storyboard`

    storyboard "Your Video File.mkv"

Storyboard will then output a file named `Your Video File.pdf` in the directory you ran the command from. With non-HD content, this process can take just a minute or two for a 22 minute file, and the output file will be around 12MB.

You can also pass in the path to a folder containing multiple video files, and Storyboard will process each one.

    $ ls "~/TV/ShowName/Season 1"
    ShowName.1x01.EpisodeName.mkv    ShowName.1x02.AnotherEpisode.mkv
    $ storyboard "~/TV/ShowName/Season 1"
    [... ... ...]
    $ ls .
    ShowName.1x01.EpisodeName.pdf    ShowName.1x02.AnotherEpisode.pdf

## Inner Workings

Storyboard has 2 phases to figuring out what's important in a video file: First, FFmpeg's ffprobe utility is used to scan for scene changes.

    $ ffprobe -show_frames -of compact=p=0 -f lavfi "movie=Your\ Video\ File.mkv,select=gt(scene\,.30)" -pretty

    media_type=video|key_frame=1|pkt_pts=2290|pkt_pts_time=91.600000|pkt_dts=2290|pkt_dts_time=91.600000|pkt_duration=1|pkt_duration_time=0.040000|pkt_pos=10492324|pkt_size=N/A|width=576|height=432|pix_fmt=rgb24|sample_aspect_ratio=1:1|pict_type=I|coded_picture_number=0|display_picture_number=0|interlaced_frame=0|top_field_first=0|repeat_pict=0|reference=0|tag:lavfi.scene_score=0.340000
    media_type=video|key_frame=1|pkt_pts=2523|pkt_pts_time=100.920000|pkt_dts=2523|pkt_dts_time=100.920000|pkt_duration=1|pkt_duration_time=0.040000|pkt_pos=11511798|pkt_size=N/A|width=576|height=432|pix_fmt=rgb24|sample_aspect_ratio=1:1|pict_type=I|coded_picture_number=0|display_picture_number=0|interlaced_frame=0|top_field_first=0|repeat_pict=0|reference=0|tag:lavfi.scene_score=0.400000

Once the timestamps, seen in the `pkt_pts_time` field, are pulled out, it starts working on subtitles. Subtitles are either extracted from the file itself, found in the same directory, or downloaded based on information gleaned from the filename. Storyboard uses SRT subtitles, which have a general format of

    32
    00:01:35,795 --> 00:01:37,763
    That man he's with...

    33
    00:01:38,198 --> 00:01:40,792
    ...is he wearing a cape?

    34
    00:01:41,201 --> 00:01:43,465
    I believe he is wearing a cape.

The timestamps (milliseconds after the comma, because the original author of SRT files was French) are combined with the timestamps that were collected from ffprobe. Once duplicates/very-near-misses are trimmed and a timeline is established, FFmpeg is again used, this time to extract the individual frames.

    ffmpeg -ss #{frame.time - 1.second} -i "Your Video File.mkv" -vframes 1 -ss #{frame.time} output.jpg

The `-ss` (seek) flags tell FFmpeg to jump to a certain timestamp, and `-vframes 1` says to extract just one frame. Any seek used after a `vframes` moves one frame at a time unti the timestamp is reached, while any seek time before jumps "close to" the timestamp. By using two seeks, one to a second before where we really want to be, and then a second to slowly seek to exactly where we want, the frame extraction is greatly sped up.

Finally, <a href="https://github.com/probablycorey/mini_magick">mini_magick</a> (which in turns uses ImageMagick's mogrify) is used to apply the subtitle text to each frame that falls in the subtitle timespans, and each frame is then added to a PDF using <a href="http://prawn.majesticseacreature.com">prawn</a>.

## A Few More Details

You can also install mkvnixtools to enable subtitle extraction straight from the video file, but this is strictly optional.

    brew install mkvtoolnix

Given a video file `Your Video File.mkv`, Storyboard would check for subtitles in 3 ways:

* Look in the same directory as the video for files named `Your Video File.mkv.srt` or `Your Video File.srt`.
* Run mkvtools and extract subtitles, if the tools are installed.
* Check several subtitle sites for a match of the exact video file, or a "best guess" on the name.

If no subtitles can be found with those methods, you can pass in the path to a SRT file using the `-s` option.

    storyboard -s ~/subtitles/my-video-file.srt "Your Video File.mkv"

Other options available are `-v` for more detailed output and `--no-scenes`, which will skip scene detection. Skipping scene detection will speed up processing by around a third, but only frames from the subtitle timings will be in the output, meaning that you could miss out on context.
