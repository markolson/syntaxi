---
title : Storyboard - Now with more GIFs (and unicode)
description: Storyboard Generates Books and GIFs From Subtitled Video Files
date: '2013-02-03'
versions:
  - name: Storyboard
    version: 0.5.0

---
To say that I'm happy with the feedback I got about Storyboard would be a wild understatement. While I can't turn the <a href="http://www.metafilter.com/124438/Storyboard-Make-video-files-into-PDF-books">nice words</a> people said about it on metafilter or elsewhere into anything, I did work hard to add or improve things people had trouble with. Setup is still pretty complicated, but I do have ideas on how to make it better - those'll be coming in another few weeks.

In the meantime, I've added to and tried to improve the <a href="https://github.com/markolson/storyboard/blob/master/INSTALL.md">setup instructions</a>.

If that doesn't work, please follow the normal <a href="https://github.com/markolson/storyboard/blob/master/INSTALL.md">setup instructions</a>.  If you have any troubles with them, please send me an email, get in touch with me <a href="http://twitter.com/mark_olson">on twitter</a>, or <a href="https://github.com/markolson/storyboard/issues?state=open">create an issue on github</a>. If you already have Storyboard, you should be able to update it:

    storyboard --update

The main issue I wanted to fix this time around was that of improper subtitles. Relative to other metadata about video files, the sites that support subtitles are pretty spotty. To work around this, you're presented with the list of all available subtitles that are returned for your video file. While the example below says that these are all subtitles that should work with this *exact* video file, the presence of the alternate pilot underscores the bad data that can come through.

    $ storyboard "Seinfeld 6x04.avi"
    There are multiple subtitles that could work with this file. Please choose one!
    All of these are subtitles made for this exact video file, so any should work.
    1: 'Seinfeld - 6x04 - The Chinese Woman_eng.srt', added 2005-11-15 00:00:00
    2: 'Seinfeld [1x00] - Alternative Pilot - Good News, Bad News - FoV -.srt', added 2005-09-04 00:00:00
    choice (default 1):

## Gifboard

 <img style="float: right; margin-left: 10px" src="/assets/media/storyboard/cape.gif" width="300px" />More than a few seemed to assume that Storyboard was used to create the GIFs from TV shows and movies that litter tumblr, or if it could support doing that. Well, it turns out that adding GIF support was pretty simple because Storyboard already knew when a line of text was on screen, how to extract those frames, and add subtitles to them. All that was left was saving the GIF itself, which also turned out to be pretty simple, using the same Imagemagick tools that Storyboard already required.

I only put a few hours of work into this, and it so it's lacking a lof of options - most notably ones that could control the size and quality of the GIF. Along with not being able to specify exact time ranges, font colors, or other options I can't think of right now, this makes the results pretty take-it-or-leave-it. Nevertheless, I can't stop laughing at the GIF on the right, and so it passes muster with me.

Instead of cramming more into Storyboard, I broke it out into a seperate applicaton: **gifboard**. Using it is pretty simple:

    $ gifboard -t "wearing a cape" "Seinfeld 6x04.avi"
<!-- more -->
After prompting for which subtitle file to use, it will show the lines that match the text you searched for if there are multiple lines. If there's just one match, it will use it without asking for confirmation. After you choose the line, it'll spit out a GIF into the directory you ran **gifboard** from.

    Multiple matches found.. pick one!
    1:  ...is he wearing a cape?
    2:  I believe he is wearing a cape.
    3:  Why was he wearing a cape?
    4:  Was my father wearing a cape?
    5:  They said you were with some guy
        who was wearing a cape.
    6:  ...just because
        they're wearing a cape.
    choice (default 1):

Gifboard is included in the Storyboard gem.

## Unicode

I'll be upfront and say that I don't understand unicode at any meaningful level. Nevertheless, I kept at it until I was able to render a subtitle file from a japanese show. I'm sure there are many issues left with it, but, it might maybe work. Please send me any feedback about how well it works for you.
