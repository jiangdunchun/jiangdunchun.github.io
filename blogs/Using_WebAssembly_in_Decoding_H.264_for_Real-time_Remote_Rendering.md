# Using WebAssembly in Decoding H.264 for Real-time Remote Rendering
*1/6/2019*

In [last blog](https://jiangdunchun.github.io/blog.html?id=WampFramework_Remotely_Invoke_the_API_in_Local_CSharp_Assembly_from_js.md), I had mentioned the images data from back-end to front-end were always larger than 1MB per second in our previous Remote Rendering Framework, because we just compressed these images separately (you can imagine we sent a picture in *.jpg format to HTML every time a texture was gotten from OpenGL). In order to solve this problem, we naturally thought of using video streaming to ease the bandwidth pressure.

H.264 is a mature video stream standard and has been widely used. We planned to insert a H.264 Encoder and Decoder in back-end and front-end respectively. Compared with the encoder, the decoder is much harder because we are decoding H.264 video stream by Javascript. For some historical reasons, Javascript is a less efficiency program language than c++ and c#. And we worried about if Javascript could decode more than 24 frames per second.

WebAssembly provides languages such as C/C++ and Rust with a compilation target so that they can run with near-native performance on the web. It is also designed to run alongside JavaScript, allowing both to work together.<sup>[1]</sup> This [article](https://www.smashingmagazine.com/2017/05/abridged-cartoon-introduction-webassembly/) explains it in details of why this technology is fast. [FFmpeg](http://ffmpeg.org/) contains a series of open-source libraries about video stream's encoding and decoding. Combining WebAssembly with FFmpeg, we can decode the video stream to YUV data in really high speed. Here is my demo:

<center><video src="blogs/Using_WebAssembly_in_Decoding_H.264_for_Real-time_Remote_Rendering/demo.mp4" width="80%" controls="controls"></video></center>

Even though this approach could apparently decrease the traffic and ensure the fluency as much as possible, it actually consumes much more hardware resources than before. The CPU occupancy of front-end device always stays in a high level when decoding the video stream. I try to resolve this problem using hardware decoding technology, and find it is an expensive project: dfferent OSs(Windows, Linux, macOS, Android) and chip makers(Intel，AMD，Nvidia) have their own solutions. I will share them when I get some achievements.

## References
[1] [https://developer.mozilla.org/en-US/docs/WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly)