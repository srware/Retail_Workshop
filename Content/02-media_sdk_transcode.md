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

## Asynchronous Transcoding
To better utilise the GPU we can make our transcode pipeline asynchronous so more than one decode and encode operation can run at once. This means for each execution of our transcode loop we submit multiple "tasks" before synchronising the pipeline.

 - First we add a parameter to our decoder parameters to tell the decoder how many tasks we want to execute asynchronously. We set this parameter initially to **1** to mimic synchronous operation. We will later increase this to see the effect it has on performance and GPU utilisation.
``` cpp
    mfxDecParams.AsyncDepth = 1;
```
 - We also need to update our encode parameters in the same way. We use the same value as we used for the decode parameters to keep things aligned.
``` cpp
	mfxEncParams.AsyncDepth = mfxDecParams.AsyncDepth;
```
 - We now need to create a task pool for our encoding operations. Rather than having a single bit stream buffer for our encoder output each "task" has it's own so we need to replace the code in **section 8** with the following:
``` cpp
    //8. Create task pool to improve asynchronous performance
    mfxU16 taskPoolSize = mfxEncParams.AsyncDepth;
    Task* pTasks = new Task[taskPoolSize];
    memset(pTasks, 0, sizeof(Task) * taskPoolSize);
    for (int i = 0; i < taskPoolSize; i++) {
        // Prepare Media SDK bit stream buffer
        pTasks[i].mfxBS.MaxLength = par.mfx.BufferSizeInKB * 1000;
        pTasks[i].mfxBS.Data = new mfxU8[pTasks[i].mfxBS.MaxLength];
        MSDK_CHECK_POINTER(pTasks[i].mfxBS.Data, MFX_ERR_MEMORY_ALLOC);
    }
```
 - In **section 9** of the code we need to add two additional variables to keep track of tasks in our transcoding loops.
``` cpp
    int nFirstSyncTask = 0;
    int nTaskIdx = 0;
```
 - We now need to modify our transcoding loops to first execute multiple tasks asynchronously and once the task pool is full synchronise the pipeline. The main transcoding loop **(Stage 1)** should now look like this:
``` cpp
    //
    // Stage 1: Main transcoding loop
    //
	while (MFX_ERR_NONE <= sts || MFX_ERR_MORE_DATA == sts || MFX_ERR_MORE_SURFACE == sts) {
		nTaskIdx = GetFreeTaskIndex(pTasks, taskPoolSize);  // Find free task
		if (MFX_ERR_NOT_FOUND == nTaskIdx) {
			// No more free tasks, need to sync
			sts = session.SyncOperation(pTasks[nFirstSyncTask].syncp, 60000);
			MSDK_CHECK_RESULT(sts, MFX_ERR_NONE, sts);

			sts = WriteBitStreamFrame(&pTasks[nFirstSyncTask].mfxBS, fSink);
			MSDK_BREAK_ON_ERROR(sts);

			pTasks[nFirstSyncTask].syncp = NULL;
			pTasks[nFirstSyncTask].mfxBS.DataLength = 0;
			pTasks[nFirstSyncTask].mfxBS.DataOffset = 0;
			nFirstSyncTask = (nFirstSyncTask + 1) % taskPoolSize;

			++nFrame;
			if (nFrame % 100 == 0) {
				printf("Frame number: %d\r", nFrame);
				fflush(stdout);
			}
		} else {
			if (MFX_WRN_DEVICE_BUSY == sts)
				MSDK_SLEEP(1);  // Wait and then repeat the same call to DecodeFrameAsync

			if (MFX_ERR_MORE_DATA == sts) {
				sts = ReadBitStreamData(&mfxBS, fSource);  // Read more data to input bit stream
				MSDK_BREAK_ON_ERROR(sts);
			}

			if (MFX_ERR_MORE_SURFACE == sts || MFX_ERR_NONE == sts) {
				nIndex = GetFreeSurfaceIndex(pSurfaces, nSurfNum);  // Find free frame surface
				MSDK_CHECK_ERROR(MFX_ERR_NOT_FOUND, nIndex, MFX_ERR_MEMORY_ALLOC);
			}
			// Decode a frame asychronously (returns immediately)
			sts = mfxDEC.DecodeFrameAsync(&mfxBS, pSurfaces[nIndex], &pmfxOutSurface, &syncpD);

			// Ignore warnings if output is available,
			// if no output and no action required just repeat the DecodeFrameAsync call
			if (MFX_ERR_NONE < sts && syncpD)
				sts = MFX_ERR_NONE;

			if (MFX_ERR_NONE == sts) {
				for (;;) {
					// Encode a frame asychronously (returns immediately)
					sts = mfxENC.EncodeFrameAsync(NULL, pmfxOutSurface, &pTasks[nTaskIdx].mfxBS, &pTasks[nTaskIdx].syncp);

					if (MFX_ERR_NONE < sts && !pTasks[nTaskIdx].syncp) { // Repeat the call if warning and no output
						if (MFX_WRN_DEVICE_BUSY == sts)
							MSDK_SLEEP(1);  // Wait if device is busy
					}
					else if (MFX_ERR_NONE < sts && pTasks[nTaskIdx].syncp) {
						sts = MFX_ERR_NONE; // Ignore warnings if output is available
						break;
					}
					else
						break;
				}

				if (MFX_ERR_MORE_DATA == sts) {
					// MFX_ERR_MORE_DATA indicates encoder need more input, request more surfaces from previous operation
					sts = MFX_ERR_NONE;
					continue;
				}
			}
		}
	}
```

 - **Stage 2** should be the same as the main transcoding loop above with the exception that we pass **NULL** to the **DecodeFrameAsync** call in order to drain the decoding pipeline.
```
sts = mfxDEC.DecodeFrameAsync(NULL, pSurfaces[nIndex], &pmfxOutSurface, &syncpD);
```

 - **Stage 3** of the transcoding process should now look like this:
```
    //
    // Stage 3: Retrieve the buffered encoded frames
    //
	while (MFX_ERR_NONE <= sts) {
		nTaskIdx = GetFreeTaskIndex(pTasks, taskPoolSize);      // Find free task
		if (MFX_ERR_NOT_FOUND == nTaskIdx) {
			// No more free tasks, need to sync
			sts = session.SyncOperation(pTasks[nFirstSyncTask].syncp, 60000);
			MSDK_CHECK_RESULT(sts, MFX_ERR_NONE, sts);

			sts = WriteBitStreamFrame(&pTasks[nFirstSyncTask].mfxBS, fSink);
			MSDK_BREAK_ON_ERROR(sts);

			pTasks[nFirstSyncTask].syncp = NULL;
			pTasks[nFirstSyncTask].mfxBS.DataLength = 0;
			pTasks[nFirstSyncTask].mfxBS.DataOffset = 0;
			nFirstSyncTask = (nFirstSyncTask + 1) % taskPoolSize;

			++nFrame;
			printf("Frame number: %d\r", nFrame);
			fflush(stdout);

		}
		else {
			for (;;) {
				// Encode a frame asychronously (returns immediately)
				sts = mfxENC.EncodeFrameAsync(NULL, NULL, &pTasks[nTaskIdx].mfxBS, &pTasks[nTaskIdx].syncp);

				if (MFX_ERR_NONE < sts && !pTasks[nTaskIdx].syncp) { // Repeat the call if warning and no output
					if (MFX_WRN_DEVICE_BUSY == sts)
						MSDK_SLEEP(1);  // Wait if device is busy
				}
				else if (MFX_ERR_NONE < sts && pTasks[nTaskIdx].syncp) {
					sts = MFX_ERR_NONE; // Ignore warnings if output is available
					break;
				}
				else
					break;
			}
		}
	}
```

 - We also need to add a **4th stage** to our transcode process in order to ensure all tasks in our task pool are synchronised and all output from the encoder gets written to disk.
```
	//
	// Stage 4: Sync all remaining tasks in task pool
	//
	while (pTasks[nFirstSyncTask].syncp) {
		sts = session.SyncOperation(pTasks[nFirstSyncTask].syncp, 60000);
		MSDK_CHECK_RESULT(sts, MFX_ERR_NONE, sts);

		sts = WriteBitStreamFrame(&pTasks[nFirstSyncTask].mfxBS, fSink);
		MSDK_BREAK_ON_ERROR(sts);

		pTasks[nFirstSyncTask].syncp = NULL;
		pTasks[nFirstSyncTask].mfxBS.DataLength = 0;
		pTasks[nFirstSyncTask].mfxBS.DataOffset = 0;
		nFirstSyncTask = (nFirstSyncTask + 1) % taskPoolSize;

		++nFrame;
		printf("Frame number: %d\r", nFrame);
		fflush(stdout);
	}
```

 - Finally we need cleanup our task buffers and task pool. Replace the following line of code:
```
    MSDK_SAFE_DELETE_ARRAY(mfxEncBS.Data);
```
with this:
```
    for (int i = 0; i < taskPoolSize; i++)
        MSDK_SAFE_DELETE_ARRAY(pTasks[i].mfxBS.Data);
    MSDK_SAFE_DELETE_ARRAY(pTasks);
```

 - **Build** the solution and run the code using the **Performance Profiler** as before. Remember we set the number of asynchronous tasks to 1 earlier so this is simply to get a benchmark before increasing the task pool size. Take a note of the **execution time** before closing the console window and take a look at the **GPU Utilization** graph.

 - Now increase the number of asynchronous operations from **1** to **4**.
``` cpp
    mfxDecParams.AsyncDepth = 4;
```

 - Once again **Build** the solution and run the **Performance Profiler**. Note the **execution time** and again take a look at the **GPU Utilization** graph. You will notice that performance has increased and the GPU is better utilised now we are performing more asynchronous operations.
