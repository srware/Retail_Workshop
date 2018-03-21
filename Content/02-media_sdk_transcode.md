# Transcoding a video stream using Intel(R) Media SDK
In this tutorial we will look at a simple transcode (decode + encode) pipeline using the Intel(R) Media SDK. We will start with a basic example using system memory for working surfaces before exploring ways of improving the performance of the transcode process using features such as opaque memory and asynchronous operation. We will also look at adding a video frame processing (VPP) resize to the transcode process.

## Getting Started

- Load the **'msdk_transcode'** Visual Studio solution file > **"Retail_Workshop\msdk_transcode\msdk_transcode.sln"**

- Once Visual Studio has loaded expand the **msdk_transcode** project in the **Solution Explorer** in the right-hand pane.

- Double-click on the **msdk_transcode.cpp** file to load the main application code.

![Open Transcode Project](images/msdk_transcode_1.jpg)

## Understanding The Code
Take a look through the existing code using the comments as a guide. This example shows the minimum API usage to transcode (decode + encode) a H.264 stream to another H.264 stream.

The basic flow is outlined below:

 1. Specify input file to decode and file to write the encoded data to
 2. Create the Intel(R) Media SDK session, decoder and encoder
 3. Configure decoder video parameters (e.g. codec)
 4. Create buffers and query parameters
	- Allocate a bit stream buffer to store encoded data before processing
	- Read the header from the input file and use this to populate the rest of the video parameters
 5. Configure encoder video parameters (e.g. codec, bitrate, rate control)
 6. Allocate the surfaces (video frame working memory) required by the decoder and encoder
 7. Initialise the Intel(R) Media SDK decode and encode components
 8. Allocate a bit stream buffer to store the output from the encoder
 9. Start the transcoding process:
	-  The first loop is the main transcoding loop where the input stream is decoded and encoded until the end of the stream is reached
	- The second loop drains the decoding pipeline once the end of the stream is reached ensuring the full input stream is encoded
	- The third loop drains the encoding pipeline once the decoding pipeline is flushed and writes the last few encoded frames to disk
 10. Clean-up resources (e.g. buffers, file handles) and end the session.

## Build & Run The Code

 - Build the solution using the keyboard shortcut **CTRL+SHIFT+B** or **Build->Build Solution** in the menu bar.
 - Make sure the application built successfully by checking the **Output** log in the bottom left pane.
 ```
 Done building project "msdk_transcode.vcxproj".
========== Build: 1 succeeded, 0 failed, 0 up-to-date, 0 skipped ==========
```
 - Run the application using the **Performance Profiler**:
	 - Select **Debug->Performance Profiler...**
	 - Make sure **CPU Usage** and **GPU Usage** are ticked and click **Start** to begin profiling.
	 
![Performance Profiler](images/msdk_transcode_2.jpg)
 
 - A console window will load running the application whilst the profiling tool records usage data in the background.
 
![Application Running](images/msdk_transcode_3.jpg)

 - Wait for the application to finish transcoding the video stream and then take note of the **execution time** printed in the console window. You can then **press 'enter' to close the command window** and stop the profiling session.
```
Frame number: 1800
Execution time: 32.84 s (54.81 fps)
Press ENTER to exit...
```
 - Now go back to Visual Studio and take a look at the **GPU Utilization** and **CPU** graphs. You will see there is limited CPU usage (assuming nothing else is happening on the system) and reasonably high GPU usage indicating that the transcode process is taking place on the GPU as expected.

![GPU Usage](images/msdk_transcode_4.jpg)

It's clear from looking at the GPU utilisation in the performance profiler output that there is a bottleneck stopping the GPU from being fully utilised by our current code. In the next section we will begin optimising the code to rectify this.