Nginx Audio Track for HTTP Live Streaming
=========

This nginx module generate audio track for hls streams on the fly.

Why?
-
Apple HTTP Live Streaming (HLS) has being adopted for almost all video stream players, and one of it's recommendations is to serve an audio-only track to users that have experiencing bad bandwidth connections.

This module aims to serve audio-only track directly on nginx, without the necessity to pre-demux the stream on Video On Demand (VoD) scenarios or the overhead and occupation of one stream output on the encoder side for live streams.

How?
-
Using a combination of nginx locations with simple scripts written in Lua and this module, it's possible to generate the entire audio track on Nginx. Look at how things are done.

A viewer requests the master playlist, and the response is modified. A simple lua script gets the first stream of the list and add an audio-playlist at the end:
<pre>
location ~ /master-playlist.m3u8$ {
    rewrite (.*)master-playlist.m3u8$ $1playlist.m3u8 break;
    content_by_lua '
        local res = ngx.location.capture(ngx.var.uri);
        local first_playlist = res.body:match("[^\\n]*m3u8")
        local audio_playlist = first_playlist:gsub("\.m3u8", "-audio.m3u8")
        local ext_inf = "#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=64000\\n"
        ngx.print(res.body)
        ngx.print(ext_inf)
        ngx.print(audio_playlist)
        ngx.print("\\n")
    ';
}
</pre>

Then, when user's connection goes bad and he needs to go to the audio target, another location will handle the request, getting the original (video) playlist and changing the extension of the chunks:
<pre>
location ~ -audio\.m3u8$ {
    default_type application/vnd.apple.mpegurl;
    content_by_lua '
        local base_m3u8_url = ngx.var.uri:gsub("-audio.m3u8", ".m3u8")
        local res = ngx.location.capture(base_m3u8_url)
        local new_body = res.body:gsub("\.ts", ".aac")
        ngx.print(new_body)
    ';
 }
</pre>

Every request for _.aac_  extensions will invoke audio extract module:
<pre>
location ~ (\.aac)$ {
    ngx_hls_audio_track;
    ngx_hls_audio_track_rootpath "/path/were/video/chunks/are/";
    expires 10m;
}
</pre>

That's it!

Status
-
This module is under development. Feedbacks, issues and patches are welcome.

Requirements
-
This module depends from some libraries (headers and shared objects) which has to be installed before it:

* avformat >= 55.0.0 (tested version: 55.0.0) - commonly distributed with [FFmpeg]
* avcodec >= 55.3.0 (tested version: 55.3.0) - commonly distributed with [FFmpeg]
* avutil >= 52.10.0 (tested version: 52.10.0) - commonly distributed with [FFmpeg]

Supported Formats
-
For now, the audio extractor module only supports extraction from _mpegts_ video chunks to _aac_ audio-only chunks.

Look at [project issues] to see which other formats are going to be supported in the future.

Installation
-
Follow the steps:

* Clone this project
<pre>
$ git clone git://github.com/flavioribeiro/nginx-audio-track-for-hls-module.git
</pre>

* Clone [Lua module]
<pre>
$ git clone git://github.com/chaoslawful/lua-nginx-module.git
</pre>

Now, download [nginx] and compile it using both modules:
<pre>
$ ./configure --add-module=/path/to/nginx-audio-track-for-hls-module --add-module=/path/to/lua-nginx-module
$ make install
</pre>

Now you can look at our [nginx configuration example] and make your changes. Have fun!

License
-

GPL. *Free Software, Fuck Yeah!*


  [nginx configuration example]: https://github.com/flavioribeiro/nginx-audio-track-for-hls-module/blob/master/nginx.conf
  [nginx]: http://nginx.org/en/download.html
  [Lua module]: https://github.com/chaoslawful/lua-nginx-module
  [FFmpeg]: http://ffmpeg.org
  [project issues]: https://github.com/flavioribeiro/nginx-audio-track-for-hls-module/issues
