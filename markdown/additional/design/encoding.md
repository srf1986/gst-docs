# Encoding and Muxing

## Problems this proposal attempts to solve

  - Duplication of pipeline code for gstreamer-based applications
    wishing to encode and or mux streams, leading to subtle differences
    and inconsistencies across those applications.

  - No unified system for describing encoding targets for applications
    in a user-friendly way.

  - No unified system for creating encoding targets for applications,
    resulting in duplication of code across all applications,
    differences and inconsistencies that come with that duplication, and
    applications hardcoding element names and settings resulting in poor
    portability.

## Goals

1.  Convenience encoding element

    Create a convenience `GstBin` for encoding and muxing several streams,
    hereafter called 'EncodeBin'.

    This element will only contain one single property, which is a profile.

2.  Define a encoding profile system

3.  Encoding profile helper library

Create a helper library to:

 - create EncodeBin instances based on profiles, and

 - help applications to create/load/save/browse those profiles.

## EncodeBin

### Proposed API

EncodeBin is a `GstBin` subclass.

It implements the `GstTagSetter` interface, by which it will proxy the
calls to the muxer.

Only two introspectable property (i.e. usable without extra API):
 - A `GstEncodingProfile`
 - The name of the profile to use

When a profile is selected, encodebin will:

 - Add REQUEST sinkpads for all the `GstStreamProfile`
 - Create the muxer and expose the source pad

Whenever a request pad is created, encodebin will:

 - Create the chain of elements for that pad
 - Ghost the sink pad
 - Return that ghost pad

This allows reducing the code to the minimum for applications wishing to
encode a source for a given profile:

```c
encbin = gst_element_factory_make ("encodebin, NULL);
g_object_set (encbin, "profile", "N900/H264 HQ", NULL);
gst_element_link (encbin, filesink);

vsrcpad = gst_element_get_src_pad (source, "src1");
vsinkpad = gst_element_get_request_pad (encbin, "video\_%u");
gst_pad_link (vsrcpad, vsinkpad);
```

### Explanation of the Various stages in EncodeBin

This describes the various stages which can happen in order to end up
with a multiplexed stream that can then be stored or streamed.

#### Incoming streams

The streams fed to EncodeBin can be of various types:

  - Video
  - Uncompressed (but maybe subsampled)
  - Compressed
  - Audio
  - Uncompressed (audio/x-raw)
  - Compressed
  - Timed text
  - Private streams

#### Steps involved for raw video encoding

0)  Incoming Stream

1)  Transform raw video feed (optional)

Here we modify the various fundamental properties of a raw video stream
to be compatible with the intersection of: \* The encoder `GstCaps` and \*
The specified "Stream Restriction" of the profile/target

The fundamental properties that can be modified are: \* width/height
This is done with a video scaler. The DAR (Display Aspect Ratio) MUST be
respected. If needed, black borders can be added to comply with the
target DAR. \* framerate \* format/colorspace/depth All of this is done
with a colorspace converter

2)  Actual encoding (optional for raw streams)

An encoder (with some optional settings) is used.

3)  Muxing

A muxer (with some optional settings) is used.

4)  Outgoing encoded and muxed stream

#### Steps involved for raw audio encoding

This is roughly the same as for raw video, expect for (1)

1)  Transform raw audo feed (optional)

We modify the various fundamental properties of a raw audio stream to be
compatible with the intersection of: \* The encoder `GstCaps` and \* The
specified "Stream Restriction" of the profile/target

The fundamental properties that can be modifier are: \* Number of
channels \* Type of raw audio (integer or floating point) \* Depth
(number of bits required to encode one sample)

#### Steps involved for encoded audio/video streams

Steps (1) and (2) are replaced by a parser if a parser is available for
the given format.

#### Steps involved for other streams

Other streams will just be forwarded as-is to the muxer, provided the
muxer accepts the stream type.

## Encoding Profile System

This work is based on:

 - The existing [GstPreset API documentation][gst-preset] system for elements

 - The gnome-media [GConf audio profile system][gconf-audio-profile]

 - The investigation done into device profiles by Arista and
   Transmageddon: [Research on a Device Profile API][device-profile-api],
   and [Research on defining presets usage][preset-usage].

### Terminology

  - Encoding Target Category A Target Category is a classification of
    devices/systems/use-cases for encoding.

Such a classification is required in order for: \* Applications with a
very-specific use-case to limit the number of profiles they can offer
the user. A screencasting application has no use with the online
services targets for example. \* Offering the user some initial
classification in the case of a more generic encoding application (like
a video editor or a transcoder).

Ex: Consumer devices Online service Intermediate Editing Format
Screencast Capture Computer

  - Encoding Profile Target A Profile Target describes a specific entity
    for which we wish to encode. A Profile Target must belong to at
    least one Target Category. It will define at least one Encoding
    Profile.

    Examples (with category): Nokia N900 (Consumer device) Sony PlayStation 3
    (Consumer device) Youtube (Online service) DNxHD (Intermediate editing
    format) HuffYUV (Screencast) Theora (Computer)

  - Encoding Profile A specific combination of muxer, encoders, presets
    and limitations.

    Examples: Nokia N900/H264 HQ, Ipod/High Quality, DVD/Pal,
    Youtube/High Quality HTML5/Low Bandwith, DNxHD

### Encoding Profile

An encoding profile requires the following information:

  - Name This string is not translatable and must be unique. A
    recommendation to guarantee uniqueness of the naming could be:
    <target>/<name>
  - Description This is a translatable string describing the profile
  - Muxing format This is a string containing the GStreamer media-type
    of the container format.
  - Muxing preset This is an optional string describing the preset(s) to
    use on the muxer.
  - Multipass setting This is a boolean describing whether the profile
    requires several passes.
  - List of Stream Profile

2.3.1 Stream Profiles

A Stream Profile consists of:

  - Type The type of stream profile (audio, video, text, private-data)
  - Encoding Format This is a string containing the GStreamer media-type
    of the encoding format to be used. If encoding is not to be applied,
    the raw audio media type will be used.
  - Encoding preset This is an optional string describing the preset(s)
    to use on the encoder.
  - Restriction This is an optional `GstCaps` containing the restriction
    of the stream that can be fed to the encoder. This will generally
    containing restrictions in video width/heigh/framerate or audio
    depth.
  - presence This is an integer specifying how many streams can be used
    in the containing profile. 0 means that any number of streams can be
    used.
  - pass This is an integer which is only meaningful if the multipass
    flag has been set in the profile. If it has been set it indicates
    which pass this Stream Profile corresponds to.

### 2.4 Example profile

The representation used here is XML only as an example. No decision is
made as to which formatting to use for storing targets and profiles.

```
<gst-encoding-target>
      <name>Nokia N900</name>
      <category>Consumer Device</category>
      <profiles>
        <profile>Nokia N900/H264 HQ</profile>
        <profile>Nokia N900/MP3</profile>
        <profile>Nokia N900/AAC</profile>
      </profiles>
    </gst-encoding-target>

    <gst-encoding-profile>
      <name>Nokia N900/H264 HQ</name>
      <description>
        High Quality H264/AAC for the Nokia N900
      </description>
      <format>video/quicktime,variant=iso</format>
      <streams>
        <stream-profile>
          <type>audio</type>
          <format>audio/mpeg,mpegversion=4</format>
          <preset>Quality High/Main</preset>
          <restriction>audio/x-raw,channels=[1,2]</restriction>
          <presence>1</presence>
        </stream-profile>
        <stream-profile>
          <type>video</type>
          <format>video/x-h264</format>
          <preset>Profile Baseline/Quality High</preset>
          <restriction>
            video/x-raw,width=[16, 800],\
	    height=[16, 480],framerate=[1/1, 30000/1001]
          </restriction>
          <presence>1</presence>
        </stream-profile>
      </streams>
    </gst-encoding-profile>
```

### API

A proposed C API is contained in the gstprofile.h file in this
directory.

### Modifications required in the existing GstPreset system

#### Temporary preset.

Currently a preset needs to be saved on disk in order to be used.

This makes it impossible to have temporary presets (that exist only
during the lifetime of a process), which might be required in the new
proposed profile system

#### Categorisation of presets.

Currently presets are just aliases of a group of property/value without
any meanings or explanation as to how they exclude each other.

Take for example the H264 encoder. It can have presets for: \* passes
(1,2 or 3 passes) \* profiles (Baseline, Main, ...) \* quality (Low,
medium, High)

In order to programmatically know which presets exclude each other, we
here propose the categorisation of these presets.

This can be done in one of two ways 1. in the name (by making the name
be `[<category>:]<name>`) This would give for example: "Quality:High",
"Profile:Baseline" 2. by adding a new `_meta` key This would give for
example: `_meta/category:quality`

#### Aggregation of presets.

There can be more than one choice of presets to be done for an element
(quality, profile, pass).

This means that one can not currently describe the full configuration of
an element with a single string but with many.

The proposal here is to extend the `GstPreset` API to be able to set all
presets using one string and a well-known separator ('/').

This change only requires changes in the core preset handling code.

This would allow doing the following: `gst_preset_load_preset
(h264enc, "pass:1/profile:baseline/quality:high")`

### Points to be determined

This document hasn't determined yet how to solve the following problems:

#### Storage of profiles

One proposal for storage would be to use a system wide directory (like
$prefix/share/gstreamer-0.10/profiles) and store XML files for every
individual profiles.

Users could then add their own profiles in ~/.gstreamer-0.10/profiles

This poses some limitations as to what to do if some applications want
to have some profiles limited to their own usage.

## Helper library for profiles

These helper methods could also be added to existing libraries (like
`GstPreset`, `GstPbUtils`, ..).

The various API proposed are in the accompanying gstprofile.h file.

### Getting user-readable names for formats

This is already provided by `GstPbUtils`.

### Hierarchy of profiles

The goal is for applications to be able to present to the user a list of
combo-boxes for choosing their output profile:

```
[ Category ] # optional, depends on the application [ Device/Site/..
] # optional, depends on the application [ Profile ]
```
Convenience methods are offered to easily get lists of categories,
devices, and profiles.

### Creating Profiles

The goal is for applications to be able to easily create profiles.

The applications needs to be able to have a fast/efficient way to: \*
select a container format and see all compatible streams he can use with
it. \* select a codec format and see which container formats he can use
with it.

The remaining parts concern the restrictions to encoder input.

### Ensuring availability of plugins for Profiles

When an application wishes to use a Profile, it should be able to query
whether it has all the needed plugins to use it.

This part will use `GstPbUtils` to query, and if needed install the
missing plugins through the installed distribution plugin installer.

## Use-cases researched

This is a list of various use-cases where encoding/muxing is being used.

### Transcoding

The goal is to convert with as minimal loss of quality any input file
for a target use. A specific variant of this is transmuxing (see below).

Example applications: Arista, Transmageddon

### Rendering timelines

The incoming streams are a collection of various segments that need to
be rendered. Those segments can vary in nature (i.e. the video
width/height can change). This requires the use of identiy with the
single-segment property activated to transform the incoming collection
of segments to a single continuous segment.

Example applications: PiTiVi, Jokosher

### Encoding of live sources

The major risk to take into account is the encoder not encoding the
incoming stream fast enough. This is outside of the scope of encodebin,
and should be solved by using queues between the sources and encodebin,
as well as implementing QoS in encoders and sources (the encoders
emitting QoS events, and the upstream elements adapting themselves
accordingly).

Example applications: camerabin, cheese

### Screencasting applications

This is similar to encoding of live sources. The difference being that
due to the nature of the source (size and amount/frequency of updates)
one might want to do the encoding in two parts: \* The actual live
capture is encoded with a 'almost-lossless' codec (such as huffyuv) \*
Once the capture is done, the file created in the first step is then
rendered to the desired target format.

Fixing sources to only emit region-updates and having encoders capable
of encoding those streams would fix the need for the first step but is
outside of the scope of encodebin.

Example applications: Istanbul, gnome-shell, recordmydesktop

### Live transcoding

This is the case of an incoming live stream which will be
broadcasted/transmitted live. One issue to take into account is to
reduce the encoding latency to a minimum. This should mostly be done by
picking low-latency encoders.

Example applications: Rygel, Coherence

### Transmuxing

Given a certain file, the aim is to remux the contents WITHOUT decoding
into either a different container format or the same container format.
Remuxing into the same container format is useful when the file was not
created properly (for example, the index is missing). Whenever
available, parsers should be applied on the encoded streams to validate
and/or fix the streams before muxing them.

Metadata from the original file must be kept in the newly created file.

Example applications: Arista, Transmaggedon

### Loss-less cutting

Given a certain file, the aim is to extract a certain part of the file
without going through the process of decoding and re-encoding that file.
This is similar to the transmuxing use-case.

Example applications: PiTiVi, Transmageddon, Arista, ...

### Multi-pass encoding

Some encoders allow doing a multi-pass encoding. The initial pass(es)
are only used to collect encoding estimates and are not actually muxed
and outputted. The final pass uses previously collected information, and
the output is then muxed and outputted.

### Archiving and intermediary format

The requirement is to have lossless

### CD ripping

Example applications: Sound-juicer

### DVD ripping

Example application: Thoggen

### Research links

Some of these are still active documents, some other not

[gst-preset]: http://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstPreset.html
[gconf-audio-profile]: http://www.gnome.org/~bmsmith/gconf-docs/C/gnome-media.html
[device-profile-api]: http://gstreamer.freedesktop.org/wiki/DeviceProfile (FIXME: wiki is gone)
[preset-usage]: http://gstreamer.freedesktop.org/wiki/PresetDesign  (FIXME: wiki is gone)

