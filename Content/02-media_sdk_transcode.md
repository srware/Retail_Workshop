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

## Opaque Memory
In our current code we are using system memory for the working surfaces used by our decoder and encoder which means frames have to be passed between system memory and video memory multiple times during the transcode pipeline limiting performance. The Intel(R) Media SDK has a feature called **Opaque Memory** which hides surface allocation specifics and allows the SDK to select the best type for execution in hardware or software. This means that if the pipeline allows, surfaces will reside in video memory for best performance. It's worth mentioning that whilst opaque memory is an easy solution for optimised surface allocation in simple situations, if you need to integrate components outside of the Intel(R) Media SDK application-level video memory allocation is required.

 - We start by updating the IO pattern in our decoder video parameters from **MFX_IOPATTERN_OUT_SYSTEM_MEMORY** to **MFX_IOPATTERN_OUT_OPAQUE_MEMORY**.
``` cpp
	mfxDecParams.IOPattern = MFX_IOPATTERN_OUT_OPAQUE_MEMORY;
```
 - We make the same change to our encoder video parameters.
``` cpp
     mfxEncParams.IOPattern = MFX_IOPATTERN_IN_OPAQUE_MEMORY;
```
 - We need to update our surface allocation code to use opaque memory allocation. Replace the code in **section 6** with the code below.
	 - Note that we don't allocate buffer memory anymore as for opaque memory this is handled internally by the SDK. We simply create **mfxExtOpaqueSurfaceAlloc** structures to hold a reference to the allocated surfaces and attach these to our decoder and encoder.
``` cpp
    //6. Initialize shared surfaces for decoder and encoder
    mfxFrameSurface1** pSurfaces = new mfxFrameSurface1 *[nSurfNum];
    MSDK_CHECK_POINTER(pSurfaces, MFX_ERR_MEMORY_ALLOC);
    for (int i = 0; i < nSurfNum; i++) {
        pSurfaces[i] = new mfxFrameSurface1;
        MSDK_CHECK_POINTER(pSurfaces[i], MFX_ERR_MEMORY_ALLOC);
        memset(pSurfaces[i], 0, sizeof(mfxFrameSurface1));
        memcpy(&(pSurfaces[i]->Info), &(DecRequest.Info), sizeof(mfxFrameInfo));
    }

    // Create the mfxExtOpaqueSurfaceAlloc structure for both encoder and decoder, the
    // allocated surfaces will be attached to these structures for the pipeline initialisations.
    mfxExtOpaqueSurfaceAlloc extOpaqueAllocDec;
    memset(&extOpaqueAllocDec, 0, sizeof(extOpaqueAllocDec));
    extOpaqueAllocDec.Header.BufferId = MFX_EXTBUFF_OPAQUE_SURFACE_ALLOCATION;
    extOpaqueAllocDec.Header.BufferSz = sizeof(mfxExtOpaqueSurfaceAlloc);
    mfxExtBuffer* pExtParamsDec = (mfxExtBuffer*) & extOpaqueAllocDec;

    mfxExtOpaqueSurfaceAlloc extOpaqueAllocEnc;
    memset(&extOpaqueAllocEnc, 0, sizeof(extOpaqueAllocEnc));
    extOpaqueAllocEnc.Header.BufferId = MFX_EXTBUFF_OPAQUE_SURFACE_ALLOCATION;
    extOpaqueAllocEnc.Header.BufferSz = sizeof(mfxExtOpaqueSurfaceAlloc);
    mfxExtBuffer* pExtParamsEnc = (mfxExtBuffer*) & extOpaqueAllocEnc;

    // Attach the surfaces to the decoder output and the encoder input
    extOpaqueAllocDec.Out.Surfaces = pSurfaces;
    extOpaqueAllocDec.Out.NumSurface = nSurfNum;
    extOpaqueAllocDec.Out.Type = DecRequest.Type;
    memcpy(&extOpaqueAllocEnc.In, &extOpaqueAllocDec.Out, sizeof(extOpaqueAllocDec.Out));

    mfxDecParams.ExtParam = &pExtParamsDec;
    mfxDecParams.NumExtParam = 1;
    mfxEncParams.ExtParam = &pExtParamsEnc;
    mfxEncParams.NumExtParam = 1;
   ```

 - Finally remove the following line from the cleanup code as we no longer manage surface buffers in the application code.
``` cpp
    MSDK_SAFE_DELETE_ARRAY(surfaceBuffers);
```

 - **Build** the solution as you did previously and again use the **Performance Profiler** to run the application. Take note of the **execution time** before closing the console window. This should be slightly improved since implementing opaque memory allocation, however, if you now look at the **GPU Utilization** graph in the performance profiler you will see the GPU remains underutilised.