HTTP upload
===========


Introduction
------------
Shaka Packager can upload produced artefacts to a HTTP server using
POST, PUT and PATCH methods to improve live publishing performance
when content is not served directly from the packaging output location.

.. seealso::

    The discussion about this feature currently happens at
    https://github.com/google/shaka-packager/issues/149,
    feel free to join.


Research
--------
We identified four sensible ways of uploading files using HTTP.

1. HTTP POST
   full upload, HTML form / multipart stream, see `RFC 1867`_

2. HTTP PUT
   full upload, see `RFC 2616 » PUT`_

3. HTTP PUT with chunked transfer
   incremental upload, see `RFC 2616 » Chunked Transfer Coding`_

4. HTTP PATCH with append
   incremental upload, see `RFC 5789`_ plus ``Update-Range: append``

Each method has different pros and cons regarding behavior, compatibility,
conveniency and efficiency. Note that while raw performance is in the main
focus, other aspects like idempotency, server-side constraints or which
mode would be suitable for live vs. non-live transmission might also be
relevant for choosing the transport method which fits your needs best.


Implementation
--------------
This section describes the current state of the implementation,
contributions are welcome.

The "HTTP PATCH with append" (4.) method has been implemented first, but the
"HTTP PUT with chunked transfer" (3.) method suggested by @colleenkhenry
and provided by @kqyang will be implemented, being an even better variant.
As @kqyang noted, this transport mode should be more efficient than
HTTP PATCH as only one HTTP connection needs to be setup.


Synopsis
--------
Here is a basic example. It is similar to the "live" example and also
borrow features from "FFmpeg piping", see :doc:`live` and :doc:`ffmpeg_piping`.

Grab and run `httpd-reflector.py`_ to use it as a dummy HTTP sink::

    wget https://gist.githubusercontent.com/amotl/3ed38e461af743aeeade5a5a106c1296/raw/httpd-reflector.py
    chmod +x httpd-reflector.py
    ./httpd-reflector.py --port 6767

Define UNIX pipe to connect ffmpeg with packager::

    export PIPE=/tmp/bigbuckbunny.fifo
    mkfifo ${PIPE}

Acquire and transcode RTMP stream::

    ffmpeg -fflags nobuffer -threads 0 -y \
        -i rtmp://184.72.239.149/vod/mp4:bigbuckbunny_450.mp4 \
        -pix_fmt yuv420p -vcodec libx264 -preset:v superfast -acodec aac \
        -f mpegts pipe: > ${PIPE}

Configure and run packager::

    # Content upload target
    export META_PATH=http://localhost:6767/hls-live/meta
    export DATA_PATH=http://localhost:6767/hls-live/media

    packager \
        "input=${PIPE},stream=audio,segment_template=${DATA_PATH}/bigbuckbunny-audio-aac-\$Number%04d\$.aac,playlist_name=bigbuckbunny-audio.m3u8,hls_group_id=audio" \
        "input=${PIPE},stream=video,segment_template=${DATA_PATH}/bigbuckbunny-video-h264-450-\$Number%04d\$.ts,playlist_name=bigbuckbunny-video-450.m3u8" \
        --io_block_size 65536 --fragment_duration 2 --segment_duration 2 \
        --time_shift_buffer_depth 3600 --preserved_segments_outside_live_window 7200 \
        --hls_master_playlist_output "${META_PATH}/bigbuckbunny.m3u8" \
        --hls_playlist_type LIVE

Output
------
The terminal running ``httpd-reflector.py`` should display
the payload chunks arriving from ``packager``.

**Main .m3u8 file**::

    ----- Request Start ----->
    Method:         PATCH
    Path:           /hls-live/meta/bigbuckbunny.m3u8

    Headers:
    Accept:         */*
    Content-Length: 360
    Content-Type:   application/octet-stream
    Host:           localhost:6767
    Update-Range:   append
    User-Agent:     shaka-packager-uploader/0.1

    Payload:
    b'#EXTM3U\n## Generated with https://github.com/google/shaka-packager version f32c934-release\n\n#EXT-X-MEDIA:TYPE=AUDIO,URI="bigbuckbunny-audio.m3u8",GROUP-ID="audio",NAME="stream_0",AUTOSELECT=YES,CHANNELS="2"\n\n#EXT-X-STREAM-INF:BANDWIDTH=134423,AVERAGE-BANDWIDTH=131947,CODECS="avc1.64000c,mp4a.40.2",RESOLUTION=320x180,AUDIO="audio"\nbigbuckbunny-video-450.m3u8\n'
    <----- Request End -----

**Auxiliary .m3u8 files for audio and video**::

    ----- Request Start ----->
    Method:         PATCH
    Path:           /hls-live/meta/bigbuckbunny-audio.m3u8

    Headers:
    Accept:         */*
    Content-Length: 216
    Content-Type:   application/octet-stream
    Host:           localhost:6767
    Update-Range:   append
    User-Agent:     shaka-packager-uploader/0.1

    Payload:
    b'#EXTM3U\n#EXT-X-VERSION:6\n## Generated with https://github.com/google/shaka-packager version f32c934-release\n#EXT-X-TARGETDURATION:1\n#EXTINF:0.939,\nhttp://localhost:6767/hls-live/media/bigbuckbunny-audio-aac-0001.aac\n'
    <----- Request End -----

    ----- Request Start ----->
    Method:         PATCH
    Path:           /hls-live/meta/bigbuckbunny-video-450.m3u8

    Headers:
    Accept:         */*
    Content-Length: 220
    Content-Type:   application/octet-stream
    Host:           localhost:6767
    Update-Range:   append
    User-Agent:     shaka-packager-uploader/0.1

    Payload:
    b'#EXTM3U\n#EXT-X-VERSION:6\n## Generated with https://github.com/google/shaka-packager version f32c934-release\n#EXT-X-TARGETDURATION:9\n#EXTINF:8.875,\nhttp://localhost:6767/hls-live/media/bigbuckbunny-video-h264-450-0001.ts\n'
    <----- Request End -----

**Audio and video data**::

    ----- Request Start ----->
    Method:         PATCH
    Path:           /hls-live/media/bigbuckbunny-audio-aac-0001.aac

    Headers:
    Accept:         */*
    Content-Length: 15775
    Content-Type:   application/octet-stream
    Expect:         100-continue
    Host:           localhost:6767
    Update-Range:   append
    User-Agent:     shaka-packager-uploader/0.1

    Payload:
    b'ID3\x04\x00\x00\x00 [...]'
    <----- Request End -----

    ----- Request Start ----->
    Method:         PATCH
    Path:           /hls-live/media/bigbuckbunny-video-h264-450-0001.ts

    Headers:
    Accept:         */*
    Content-Length: 65536
    Content-Type:   application/octet-stream
    Expect:         100-continue
    Host:           localhost:6767
    Update-Range:   append
    User-Agent:     shaka-packager-uploader/0.1

    Payload:
    b'G@P<\x07\x10\x00\x03\x9 [...]'
    <----- Request End -----


----

Have fun!


.. _RFC 1867: https://tools.ietf.org/html/rfc1867
.. _RFC 2616 » PUT: https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.6
.. _RFC 2616 » Chunked Transfer Coding: https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1
.. _RFC 5789: https://tools.ietf.org/html/rfc5789
.. _httpd-reflector.py: https://gist.github.com/amotl/3ed38e461af743aeeade5a5a106c1296

