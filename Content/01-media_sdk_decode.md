# Decoding a video stream using Intel(R) Media SDK
In this tutorial you will learn the basic principles behind decoding a video stream using the Intel(R) Media SDK. You will understand how to configure the Intel(R) Media SDK pipeline to decode a 4K 30fps AVC stream initially using a software decode implementation and then optimising the code to utilise hardware based decoding.

## Getting Started

- Load the **'msdk_decode'** Visual Studio solution file > **"Retail_Workshop\msdk_decode\msdk_decode.sln"**

- Once Visual Studio has loaded expand the **msdk_decode** project in the **Solution Explorer** in the right-hand pane.

- Double-click on the **msdk_decode.cpp** file to load the main application code.

![Open Decode Project](images/msdk_decode_1.jpg)

## Understanding The Code
Take a look through the existing code using the comments as a guide. This example shows the minimum API usage to decode a H.264 stream.

The basic flow is outlined below:

 1. Specify input file to decode
 2. Initialise a new Media SDK session and decoder
 3. Configure basic video parameters (e.g. codec)
 4. Create buffers and query parameters
	- Allocate a bit stream buffer to store encoded data before processing
	- Read the header from the input file and use this to populate the rest of the video parameters
	- Run a query to check for the validity and SDK support of these parameters
 5. Allocate the surfaces (video frame working memory) required by the decoder
 6. Initialise the decoder
 7. Start the decoding process:
	- The first loop runs until the entire stream has been decoded
	- The second loop drains the decoding pipeline once the end of the stream is reached
8. Clean-up resources (e.g. buffers, file handles) and end the session.

## Build & Run The Code

 - Build the solution: **Build->Build Solution**
 
![Build Solution](images/msdk_decode_2.jpg)
 - Make sure the application built successfully by checking the **Output** log in the bottom left pane.
 
![Check Build](images/msdk_decode_3.jpg)
 - Run the application using the **Performance Profiler**:
	 - Select **Debug->Performance Profiler...**
	 - Make sure **CPU Usage** and **GPU Usage** are ticked and click **Start** to begin profiling.
	 
![Performance Profiler](images/msdk_decode_4.jpg)
 - A console window will load running the application whilst the profiling tool records usage data in the background.
 
![Application Running](images/msdk_decode_5.jpg)
 - Wait for the application to finish decoding the video stream and then take note of the **execution time** printed in the console window. You can then **press 'enter' to close the command window** and stop the profiling session.
 
![Stop Application](images/msdk_decode_6.jpg)
 - Now go back to Visual Studio and take a look at the **GPU Utilization** and **CPU** graphs. Notice that CPU usage is high and GPU usage is low confirming that CPU based decoding is taking place.

![CPU Usage](images/msdk_decode_7.jpg)

## Hardware Decoding
Where possible we want to use hardware based decoding for improved efficiency and speed. The Intel(R) Media SDK is able to select the best decode implementation based on the platform capabilities, first checking to see if hardware can be used and falling back to software if not.

 - Change the Media SDK implementation from **'MFX_IMPL_SOFTWARE'** to **'MFX_IMPL_AUTO_ANY**:

``` cpp
    mfxIMPL impl = MFX_IMPL_AUTO_ANY;
```

 - Rebuild the project and once again run the **Performance Profiler** as before. Once again take note of the **execution time** before closing the console window. You will now see that **GPU** utilisation is high and **CPU** usage is low (depending on what else is happening on the system).
 - Click on **'View Details'** in the **GPU Usage** window below the graphs to get a better breakdown of GPU engine utilisation (3D and Video Decode). Note the heavy utilisation of the **GPU VIDEO_DECODE** engine.

![GPU Details](images/msdk_decode_8.jpg)
# Decoding a video stream using Intel(R) Media SDK
In this tutorial you will learn the basic principles behind decoding a video stream using the Intel(R) Media SDK. You will understand how to configure the Intel(R) Media SDK pipeline to decode a 4K 30fps AVC stream initially using a software decode implementation and then optimising the code to utilise hardware based decoding.

## Getting Started

- Load the **'msdk_decode'** Visual Studio solution file > **"Retail_Workshop\msdk_decode\msdk_decode.sln"**

- Once Visual Studio has loaded expand the **msdk_decode** project in the **Solution Explorer** in the right-hand pane.

- Double-click on the **msdk_decode.cpp** file to load the main application code.

![Open Decode Project](images/msdk_decode_1.jpg)

## Understanding The Code
Take a look through the existing code using the comments as a guide. This example shows the minimum API usage to decode a H.264 stream.

The basic flow is outlined below:

 1. Specify input file to decode
 2. Initialise a new Media SDK session and decoder
 3. Configure basic video parameters (e.g. codec)
 4. Create buffers and query parameters
	- Allocate a bit stream buffer to store encoded data before processing
	- Read the header from the input file and use this to populate the rest of the video parameters
	- Run a query to check for the validity and SDK support of these parameters
 5. Allocate the surfaces (video frame working memory) required by the decoder
 6. Initialise the decoder
 7. Start the decoding process:
	- The first loop runs until the entire stream has been decoded
	- The second loop drains the decoding pipeline once the end of the stream is reached
8. Clean-up resources (e.g. buffers, file handles) and end the session.

## Build & Run The Code

 - Build the solution: **Build->Build Solution**
 
![Build Solution](images/msdk_decode_2.jpg)
 - Make sure the application built successfully by checking the **Output** log in the bottom left pane.
 
![Check Build](images/msdk_decode_3.jpg)
 - Run the application using the **Performance Profiler**:
	 - Select **Debug->Performance Profiler...**
	 - Make sure **CPU Usage** and **GPU Usage** are ticked and click **Start** to begin profiling.
	 
![Performance Profiler](images/msdk_decode_4.jpg)
 - A console window will load running the application whilst the profiling tool records usage data in the background.
 
![Application Running](images/msdk_decode_5.jpg)
 - Wait for the application to finish decoding the video stream and then take note of the **execution time** printed in the console window. You can then **press 'enter' to close the command window** and stop the profiling session.
 
![Stop Application](images/msdk_decode_6.jpg)
 - Now go back to Visual Studio and take a look at the **GPU Utilization** and **CPU** graphs. Notice that CPU usage is high and GPU usage is low confirming that CPU based decoding is taking place.

![CPU Usage](images/msdk_decode_7.jpg)

## Hardware Decoding
Where possible we want to use hardware based decoding for improved efficiency and speed. The Intel(R) Media SDK is able to select the best decode implementation based on the platform capabilities, first checking to see if hardware can be used and falling back to software if not.

 - Change the Media SDK implementation from **'MFX_IMPL_SOFTWARE'** to **'MFX_IMPL_AUTO_ANY**:

``` cpp
    mfxIMPL impl = MFX_IMPL_AUTO_ANY;
```

 - Rebuild the project and once again run the **Performance Profiler** as before. Once again take note of the **execution time** before closing the console window. You will now see that **GPU** utilisation is high and **CPU** usage is low (depending on what else is happening on the system).
 - Click on **'View Details'** in the **GPU Usage** window below the graphs to get a better breakdown of GPU engine utilisation (3D and Video Decode).

![GPU Details](images/msdk_decode_8.jpg)

 - Note the heavy utilisation of the **GPU VIDEO_DECODE** engine.

![GPU Details](images/msdk_decode_9.jpg)
