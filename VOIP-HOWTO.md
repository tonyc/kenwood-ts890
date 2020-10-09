# Kenwood TS-890 LAN VOIP HOWTO

This file documents the Kenwood TS-890's UDP VOIP data stream and data formats.

This file is very much a work in progress!

## Enabling VOIP

First, establish a connection to the radio as described in the LAN-HOWTO in this repo.

Once connected and signed in, send `##VP1;` to enable the high-quality data stream, or `##VP2;` for the low-quality stream. Send `##VP0;` to turn it off.

After sending `##VP`, the TS-890 will immediately startstreaming UDP packets back to the connecting IP address on port `60001`

## Data Format

The packets returned are RTP (https://tools.ietf.org/html/rfc3550) packets, with he RTP payload type is specified as `96`, which means that it's a vendor-specific payload.

The payload of these packets are simply raw PCM samples, and you can pipe them into `ffmpeg` for transcoding to other formats, like MP3 or M4A (see below)

The high-quality stream is 16-bit signed, little-endian PCM samples at 16000 hz sample rate. The RTP payload will be 640 bytes long.

I think the low-quality stream is 8 bit unsigned, LE PCM at 8000 hz sample rate, but have not verified this yet. 

If you have a Wireshark capture of the UDP packets coming FROM the TS-890, and export just the raw data to a file called `input.bin`,
the following Ruby program can be used to strip the RTP headers, leaving just the raw PCM samples. This works for the high-quality stream, and
is untested with the low-quality stream.

You can then open `output.bin` in a file like Audacity, specify 16-bit LE PCM as the format, and hear audio!

```ruby
#!/usr/bin/env ruby

file = "input.bin"
outfile = "output.bin"

File.open(file) do |file|
  File.open(outfile, "wb") do |outfile|

    while (buffer = file.read(652)) do
      puts "read #{buffer.length} bytes"

      stripped_data = buffer[12, 640] # chop off 12 bytes of RTP headers we don't want
      puts "stripped down to #{stripped_data.length} bytes"

      outfile.write(stripped_data)
    end

  end
end
```

## Receiving the VOIP stream with ffmpeg

If you have a separate program that will sign in to the radio and enable VOIP, the following command can be used to output an MP3 file of the audio:

`ffmpeg -loglevel quiet -protocol_whitelist file,crypto,rtp,udp -y -vn -dn -acodec pcm_s16le -i ts890-high.sdp -ac 1 -ar 44100  out.mp3`

The `ts890-high.sdp` is required for ffmpeg to recognize the custom payload type (96). It's available in this code repository.