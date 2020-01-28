#Dashsink for a complete offering in GStreamer


##A brief introduction to adaptive streaming and MPEG DASH

Adaptive streaming is a technique to provide flexibility and scalability by offering variable bit-rate streams to the client.
Designed to work over HTTP, it provides media content as separate streams with media type and various bit-rates, the client will be able to select according to its network bandwidth or its CPU power.

The most common adaptive streaming solution are:

 * HLS (Apple HTTP Live Streaming)
 * MSS (Microsoft Smooth Streaming)
 * ADS (Adobe HTTP Dynamic Streaming)
 * MPEG-DASH (MPEG Dynamic Adaptive Streaming over HTTP)


MPEG DASH standing for Dynamic Adaptive Streaming Solution is the most complete adaptive streaming technique. This format is based on a XML description file named MPD (Media Presentation Description). This format will describe a set of `representations` for a given media type with various available bit-rate or media format.

This solution is an open format and is widely supported by the industry. For more information about it, you can visit the [DASHIF website](https://dashif.org/)

In the example below, the MPD describes a static content with three media content type (adaptation sets). Each adaptations sets contains representations. 5 representations for the video allows to switch to 5 different bit rates according to the playback constraints.

```
<MPD mediaPresentationDuration="PT634.566S" minBufferTime="PT2.00S" profiles="urn:hbbtv:dash:profile:isoff-live:2012,urn:mpeg:dash:profile:isoff-live:2011" type="static" xmlns="urn:mpeg:dash:schema:mpd:2011" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="urn:mpeg:DASH:schema:MPD:2011 DASH-MPD.xsd">
 <BaseURL>./</BaseURL>
 <Period>
  <AdaptationSet id="1" mimeType="video/mp4" contentType="video" subsegmentAlignment="true" subsegmentStartsWithSAP="1" par="16:9">
   <SegmentTemplate duration="120" timescale="30" media="$RepresentationID$/$RepresentationID$_$Number$.m4v" startNumber="1" initialization="$RepresentationID$/$RepresentationID$_0.m4v"/>
   <Representation id="bbb_30fps_1280x720_4000k" codecs="avc1.64001f" bandwidth="4952892" width="1280" height="720" frameRate="30" sar="1:1" scanType="progressive"/>
   <Representation id="bbb_30fps_320x180_200k" codecs="avc1.64000d" bandwidth="254320" width="320" height="180" frameRate="30" sar="1:1" scanType="progressive"/>
   <Representation id="bbb_30fps_480x270_600k" codecs="avc1.640015" bandwidth="759798" width="480" height="270" frameRate="30" sar="1:1" scanType="progressive"/>
   <Representation id="bbb_30fps_640x360_800k" codecs="avc1.64001e" bandwidth="1013310" width="640" height="360" frameRate="30" sar="1:1" scanType="progressive"/>
   <Representation id="bbb_30fps_3840x2160_12000k" codecs="avc1.640033" bandwidth="14931538" width="3840" height="2160" frameRate="30" sar="1:1" scanType="progressive"/>
  </AdaptationSet>

  <AdaptationSet id="2" mimeType="audio/mp4" contentType="audio" subsegmentAlignment="true" subsegmentStartsWithSAP="1">
   <Accessibility schemeIdUri="urn:tva:metadata:cs:AudioPurposeCS:2007" value="6"/>
   <Role schemeIdUri="urn:mpeg:dash:role:2011" value="main"/>
   <SegmentTemplate duration="192512" timescale="48000" media="$RepresentationID$/$RepresentationID$_$Number$.m4a" startNumber="1" initialization="$RepresentationID$/$RepresentationID$_0.m4a"/>
   <Representation id="bbb_a64k" codecs="mp4a.40.5" bandwidth="67071" audioSamplingRate="48000">
    <AudioChannelConfiguration schemeIdUri="urn:mpeg:dash:23003:3:audio_channel_configuration:2011" value="2"/>
   </Representation>
  </AdaptationSet>

  <AdaptationSet id="3" mimeType="image/jpeg" contentType="image">
    <SegmentTemplate media="$RepresentationID$/tile_$Number$.jpg" duration="100" startNumber="1"/>
    <Representation bandwidth="12288" id="thumbnails_320x180" width="3200" height="180">
      <EssentialProperty schemeIdUri="http://dashif.org/thumbnail_tile" value="10x1"/>
    </Representation>
  </AdaptationSet>

 </Period>
</MPD>
```

## DASH in GStreamer

Since 2012, GStreamer offers a DASH support only as a client solution with the [dashdemux](https://gstreamer.freedesktop.org/documentation/dashdemux/index.html?gi-language=c) whereas HLS provides the both elements, demuxer and sink since 2017.

### DASH demuxer

This element landed in the GStreamer repository in 2012 and had evolved a lot along the time to support the various use case DASH offers in its specifications and its applications. Indeed he is able to support multiple streams (video/audio/subtitles) and allows the user to select from the available streams or automatically select the best representation according to the network.

### DASH Sink

A first attempt to merge a DASH sink has been attempted in 2016. Apparently the design was not clearly mature enough to be landed.

Following the HLSSink2 design based on the element splitmuxsink, I decided to finally proposed another approach for this missing brick in GStreamer.

#### MPD Parser

In order to unify the MPD support, a first task has been to relocate and redesign the base classes to read, and write, an MPD file. Based on XML, the Media Presentation Description scheme is based on multiple nodes owning children and properties as described above in the MPD example.
A first important work was to split the code in 'object' identifying each XML node the MPD scheme can support, including the root node, periods , adaptation sets etc. An object oriented approach has been elected to unify the work regarding the parsing, object property manipulation and the XML format generation.

#### The sink

Inspired from the work on [hlssink2](https://gstreamer.freedesktop.org/documentation/hls/hlssink2.html?gi-language=c), the dash sink is a "super bin" and implements internally a [splitmuxsink](https://gstreamer.freedesktop.org/documentation/multifile/splitmuxsink.html?gi-language=c) to provide the multiple media segments. Most of the challenge, here was to write the MPD compliant with the DASHIF conformance test [here](https://conformance.dashif.org/) on time with usable and suitable media segments.

This plugin is now capable:

 * Multiple input audio/video streams
 * Multiple periods
 * TS segment supported (MP4 is more unstable), need additional work for the segment transition in short segment scheme.
 * Fragment segment with given duration
 * Static/Dynamic MPD (Passing DashIF conformance test)

#### An example is better than hundred words

In the following pipeline a static mpd file is created in `/tmp` along with one single segment long for a single video stream during a period of 60s. The segment will be encoded as h264 and encapsulated in
```
# gst-launch-1.0 -m dashsink name=dashsink mpd-root-path=/tmp target-duration=60 dynamic=false period-duration=60000 muxer=ts  v4l2src ! video/x-raw,framerate=30/1,width=320,height=240 ! videoconvert ! queue ! x264enc bitrate=400 ! dashsink.video_0
```

If you would like to learn more about `dashsink` or any other parts of GStreamer, please [contact us](https://www.collabora.com/contact-us.html)
