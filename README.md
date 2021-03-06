rgain \-- ReplayGain tools and Python library
=============================================

**WARNING - I don't fully understand the nuances of python3
string encoding and filenames. If your system doesn't use UTF-8 filename
encoding, proceed with caution!**

This Python package provides modules to read, write and calculate Replay
Gain as well as 2 scripts that utilize these modules to do Replay Gain.

Replay Gain \[1\] is a proposed standard (and has been for some time \--
but it\'s widely accepted) that\'s designed to solve the problem of
varying volumes between different audio files. I won\'t lay it all out
for you here, go read it yourself.

\[1\] <http://replaygain.org>

NOTE: rgain is currently not being developed; for more information or if
you\'d like to help remedying this situation, see:
<https://bitbucket.org/fk/rgain/issues/26/wanted-new-maintainer>

Requirements
============

-   Python 3 \-- <http://python.org/>
-   Mutagen \-- <http://code.google.com/p/mutagen/>
-   GStreamer 1.0 \-- <http://gstreamer.org/>
-   PyGObject \-- <https://live.gnome.org/PyGObject>
-   python-docutils for the manpages \--
    <http://docutils.sourceforge.net/>

To install these dependencies on Debian or Ubuntu (12.10 or newer):

    # apt-get install python python-mutagen python-docutils python-gi \
      gir1.2-gstreamer-1.0 libgstreamer1.0-0 gstreamer1.0-plugins-base \
      gstreamer1.0-plugins-good

You will also need GStreamer decoding plugins for any audio formats you
want to use.

Installation
============

Just install it like any other Python package: unpack, then (as
root/with **sudo**):

    # python setup.py install

**replaygain**
==============

This is a program like, say, **vorbisgain** or **mp3gain**, the
difference being that instead of supporting a mere one format, it
supports several:

-   Ogg Vorbis (or probably anything you can put into an Ogg container)
-   Flac
-   WavPack
-   MP4 (commonly using the AAC codec)
-   MP3

Basic usage is simple:

    $ replaygain AUDIOFILE1 AUDIOFILE2 ...

There are some options; see them by running:

    $ replaygain --help

**collectiongain**
==================

This program is designed to apply Replay Gain to whole music
collections, plus the ability to simply add new files, run
**collectiongain** and have it replay-gain those files without asking
twice.

To use it, simply run:

    $ collectiongain PATH_TO_MUSIC

and re-run it whenever you add new files. Run:

    $ collectiongain --help

to see possible options.

If, however, you want to find out how exactly **collectiongain** works,
read on (but be warned: It\'s long, boring, technical, incomprehensible
and awesome). **collectiongain** runs in two phases: The file collecting
phase and the actual run. Prior to analyzing any audio data,
**collectiongain** gathers all audio files in the directory and
determines a so-called album ID for each from the file\'s tags:

-   If the file contains a Musicbrainz album ID, that is used.
-   Otherwise, if the file contains an *album* tag, it is joined with
    either

    -   a MusicBrainz album artist ID, if that exists
    -   an *albumartist* tag, if that exists,
    -   or the *artist* tag
    -   or nothing if none of the above tags exist.

    The resulting artist-album combination is the album ID for that
    file.

-   If the file doesn\'t contain a Musicbrainz album ID or an *album*
    tag, it is presumed to be a single track without album; it will only
    get track gain, no album gain.

Since this step takes a relatively long time, the album IDs are cached
between several runs of **collectiongain**. If a file was modified or a
new file was added, the album ID will be (re-)calculated for that file
only. The program will also cache an educated guess as to whether a file
was already processed and had Replay Gain added \-- if
**collectiongain** thinks so, that file will totally ignored for the
actual run. This flag is set whenever the file is processed in the
actual run phase (save for dry runs, which you can enable with the
**\--dry-run** switch) and is cleared whenever a file was changed. You
can pass the **ignore-cache** switch to make **collectiongain** totally
ignore the cache; in that case, it will behave as if no cache was
present and read your collection from scratch.

For the actual run, **collectiongain** will simply look at all files
that have survived the cleansing described above; for files that don\'t
contain Replay Gain information, **collectiongain** will calculate it
and write it to the files (use the **\--force** flag to calculate gain
even if the file already has gain data). Here comes the big moment of
the album ID: files that have the same album ID are considered to be one
album (duh) for the calculation of album gain. If only one file of an
album is missing gain information, the whole album will be recalculated
to make sure the data is up-to-date.

MP3 formats
===========

Proper Replay Gain support for MP3 files is a bit of a mess: on the one
hand, there is the **mp3gain** application \[1\] which was relatively
widely used (I don\'t know if it still is) \-- it directly modifies the
audio data which has the advantage that it works with pretty much any
player, but it also means you have to decide ahead of time whether you
want track gain or album gain. Besides, it\'s just not very elegant. On
the other hand, there are at least two commonly used ways to store
proper Replay Gain information in ID3v2 tags \[2\].

Now, in general you don\'t have to worry about this when using this
package: by default, **replaygain** and **collectiongain** will read and
write Replay Gain information in the two most commonly used formats.
However, if for whatever reason you need more control over the MP3
Replay Gain information, you can use the **\--mp3-format** option
(supported by both programs) to change the behaviour. Possible choices
with this switch are:

*replaygain.org* (alias: *fb2k*)

:   Replay Gain information is stored in ID3v2 TXXX frames. This format
    is specified on the replaygain.org website as the recommended format
    for MP3 files. Notably, this format is also used by the foobar2000
    music player for Windows \[3\].

*legacy* (alias: *ql*)

:   Replay Gain information is stored in ID3v2.4 RVA2 frames. This
    format is described as \"legacy\" by replaygain.org; however, it is
    still the primary format for at least the Quod Libet music player
    \[4\] and possibly others. It should be noted that this format does
    not support volume adjustments of more than 64 dB: if the calculated
    gain value is smaller than -64 dB or greater than or equal to +64
    dB, it is clamped to these limit values.

*default*

:   This is the default implementation used by both **replaygain** and
    **collectiongain**. When writing Replay Gain data, both the
    *replaygain.org* as well as the *legacy* format are written. As for
    reading, if a file contains data in both formats, both data sets are
    read and then compared. If they match up, that Replay Gain
    information is returned for the file. However, if they don\'t match,
    no Replay Gain data is returned to signal that this file does not
    contain valid (read: consistent) Replay Gain information.

\[1\] <http://mp3gain.sourceforce.net>

\[2\]
<http://wiki.hydrogenaudio.org/index.php?title=ReplayGain_specification#ID3v2>

\[3\] <http://foobar2000.org>

\[4\] <http://code.google.com/p/quodlibet>

Changes
=======

rgain 2.0.0 (2020-08-30)
------------------------

-   Port to python 3.
-   Possibly break everything because string encoding is hard.

rgain 1.3.4 (2016-05-06)
------------------------

-   Support an additional format of MusicBrainz tags for MP3 files.
-   Album artist tags for MP3 files are now case-sensitive.
-   Fix a potential bug where custom reference levels would still store
    the default reference level (89 dB).
-   Update readme and PyPI description re: inactivity.

rgain 1.3.3 (2014-10-09)
------------------------

-   Fixed swapped album gain and track peak tags.

rgain 1.3.2 (2014-05-24)
------------------------

-   Fixed some problems with non-UTF8 file names. They should now work
    as long as any file names touched by **rgain** match the system
    encoding.
    (<https://bitbucket.org/fk/rgain/issue/12/unicodedecodeerror-ascii-codec-cant-decode>)
-   Misc. bug fixes.

rgain 1.3.1 (2013-11-29)
------------------------

-   Support MP4/AAC (courtesy of Yevgeny Bezman).

rgain 1.3 (2013-10-28)
----------------------

-   Work around a bug in some pygobject 3.10 releases
    (<https://bugzilla.gnome.org/show_bug.cgi?id=710447>)
-   Properly recognise file extensions even with different
    capitalisation.
-   Overhaul album ID algorithm to be hopefully better at grouping files
    that belong to an album and, conversely, not mis-grouping files.
    Note that this change will invalidate any cache files you might
    still have so your entire collection will be re-scanned next time
    you run **collectiongain**.
-   Assorted bug fixes.

rgain 1.2.1 (2013-10-18)
------------------------

-   Fix issue with reading MP3 reference loudness tags.

rgain 1.2 (2013-05-04)
----------------------

-   Port to GStreamer 1.0.
-   Support default GStreamer command-line options for **replaygain**
    and **collectiongain**. All known GStreamer options can be listed by
    using the **\--help-gst** flag.

Copyright
=========

With the exception of the manpages, all files are:

    Copyright (c) 2009-2015 Felix Krull <f_krull@gmx.de>

The manpages were originally written for the Debian project and are:

    Copyright (c) 2011 Simon Chopin <chopin.simon@gmail.com>
    Copyright (c) 2012-2015 Felix Krull <f_krull@gmx.de>
