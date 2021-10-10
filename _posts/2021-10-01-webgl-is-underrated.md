---
title: WebGL is Underrated  
published: true
tags: WebGL Unity Deep-Dive Guide Design Lessons
---

**TL;DR**  
The following article shares the learnings I’ve had getting our game up and running on the WebGL platform. With WebGL you open doors to a much wider audience since it runs on all modern browsers and so is already cross platform. You may consider WebGL if yours is not a highly CPU, GPU, Memory intensive game. Well, there are a couple of trade-offs you’ll have to make before you decide to move to WebGL. I hope by the end of this report you’ll have a better clarity on whether WebGL is for you or not.
 
Our game had been sailing smoothly on Facebook Gameroom until it heard the platform’s retirement news. Although our game is available on Android and IOS as well, but it’s desktop experience was too good for us to sacrifice. To cater to our desktop players we started exploring other platforms and that is when UWP and WebGL came under our radar. Of Course UWP made more sense to explore first since it was the closest to the Gameroom platform but due to the lack of facebook connectivity on UWP (no support from facebook sdk for unity) and official IAP support we began exploring WebGL which eventually worked for us.

### Contents

1. Why WebGL
2. Cons of the WebGL platform  
  2.1 Lack of threading  
  2.2 Memory Management  
  2.3 Lack of raw TCP/UDP sockets
3. Build Process
4. Dev Insights  
  4.1 Platform dependant preprocessor directive  
  4.2 Cross Origin Resource Sharing (CORS)  
  4.3 Memory Considerations  
  4.4 Threads to Coroutines  
  4.5 Local State Management  
  4.6 Localisation on WebGL  
  4.7 Calling Custom Javascript with C# Adapter  
  4.8 Measuring and Debugging Runtime Memory  
  4.9 Caching with Indexed DB  
  4.10 Using the right animation API on browser for better performance  
  4.11 Analysing Unity Build Reports  
  4.12 Using WebAssembly  
  4.13 WebGL Templates  
  4.14 Publishing Platform  
5. Conclusion

## Why should you even consider WebGL as a target platform?

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl1.gif">
</p>

The answer is pretty straightforward, it runs on all modern browsers so your game is already cross platform with which you can reach a much wider audience. Also graphics performance on WebGL is pretty close to native apps because its graphics API uses your GPU for hardware-accelerated rendering. Also it is much easier on web platforms to control what version of your app users use on an everyday basis.

## Not so great side of WebGL

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl2.gif">
</p>

There is a lot you can do with WebGL but at the end of the day it is managed by the browser which has its own set of limitations:

1. **Lack of threading**  
It is not possible to spawn threads to leverage multiple CPU cores for parallel processing on browsers. Instead browsers use coroutines and run everything in a single thread to achieve concurrency which is good for I/O intensive workloads. 
If your game features are okay with this then you can convert your threads to coroutines and have your game running on WebGL with a slight compromise on performance.

2. **Memory Management**  
Unlike native apps you get very little control over how memory is managed on web browsers. Unity creates a heap of fixed size (as specified by the developer) in memory which requires a contiguous block. It uses this heap to store all its runtime objects.
Okay hold, there are a couple of problems here:  
    &nbsp;&nbsp;2.1 Max size of this heap cannot exceed 2GB because beyond that 32-bit systems will overflow  
    &nbsp;&nbsp;2.2 This heap has to be a contiguous block of memory, so if the browser fails to allocate then your game will crash  
    &nbsp;&nbsp;2.3 If at runtime your game tries to allocate more memory than the size of the heap, again the game will crash with OOM exception  
    &nbsp;&nbsp;2.4 This heap size has to be predetermined at build time which can get a little tricky to figure out.  
    &nbsp;&nbsp;2.5 Garbage collection happens after a frame completes, it doesn’t get a chance to run between the frames, so it increases the chances of overflow if there is a memory intensive operation in a single frame cycle

3. **Lack of raw TCP/UDP sockets on browsers**   
This is not that big a problem to be honest but adds on to the development effort. There are wrappers available that will help you convert your TCP code to Websockets which is supported by browsers. Websockets is an application layer protocol built on top of TCP so please expect some perf drop when using websockets compared to raw TCP sockets. 
Another application layer protocol supported by browsers is WebRTC used for P2P communication. It supports both TCP and UDP but sees most of its application in audio/video streaming. You can try to leverage WebRTC but it's a risky territory since the protocol is quite complex.
For our game we were already using HTTP for all our communication so we didn’t have to get our hands dirty here.  

## Build Process

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl3.png">
</p>

Build process for WebGL on Unity is pretty straight forward. Since IL2CPP is the only supported backend for WebGL, C# code after running through preprocessor and C# compiler gives out Intermediate Language (IL) which is later converted to C++ using IL2CPP. This is again transpiled into javascript (asm.js) using Emscripten which is executable on browsers. But if the chosen linker target is webassembly then c++ is directly compiled into wasm.

## 750M Headstart

<p align="center">
  <img width="580" height="380" src="{{site.baseurl}}/assets/images/webgl4.gif">
</p>

Okay, if you think WebGL can work for you then you can leverage some of the learnings I’ve had getting our game out on WebGL:

### **Platform dependant preprocessor directive**  
`UNITY_WEBGL` is platform dependent macro which will help you segregate some of the platform specific logic.
```
# if UNITY_WEBGL

            // WebGL specific logic
# endif
```

### **Cross Origin Resource Sharing (CORS)**  
For security reasons, browsers restrict cross origin HTTP requests. If your client is hosted on facebook for example and your client tries to communicate with your API server hosted on api.mygameserver.com then this other server needs to whitelist facebook in its CORS policy by specifying an HTTP header `Access-Control-Allow-Origin`  
For us, our game client interacts with multiple endpoints. Our backend servers for the game logic through the PBR, S3 and Akamai for static assets. So we had to get this setting configured on all these places for all the different environments we support.

<p align="center">
  <img width="600" height="300" src="{{site.baseurl}}/assets/images/webgl5.png">
</p>

### **Memory Considerations**  
It can be hard to find the sweet spot for how much memory you should allocate to your heap in Unity which can be really frustrating. It is recommended to keep the heap size below 512MB. 
But there exists a way you can have a dynamic heap size which will automatically resize itself at runtime according to the application memory requirements. You can enable this programmatically by adding the following snippet to your C# script:
```
PlayerSettings.WebGL.emscriptenArgs = “-s ALLOW_MEMORY_GROWTH=1”;
```
This may sound like a silver bullet to this problem but trust me it is not. Emscripten will skip a lot of optimisations if you enable this option which will impact the performance of the game. You can still run into OOM with this option enabled if your system runs out of memory when growing this heap. This option is usually not recommended but seen as a last resort to problems like these.
The recommended option is to analyse your build report and optimise parts of your system that take up most of the memory. Please note this is the case for linker target asm.js on unity versions less than 2018. Webassembly is the new standard now, please use it if your unity version supports that along with the above mentioned emscripten argument which will make the wasm heap dynamic.

### **Threads to Coroutines**  
Since threading is not supported by javascript, the only option is to convert your threads to coroutines on your WebGL platform. You can use the UNITY_WEBGL macro to segregate threads and coroutines. 
The amount of hit you take on performance depends on the kind of tasks you perform in your threads. If your threaded tasks are more I/O bound then impact on performance will be less compared to a case where the tasks are long and CPU intensive.

### **Local State Management**  
You might already be using Unity’s PlayerPrefs API for local state management and so were we. We would store the user's ID here so that the next time that player returns, we identify and load the progress so that they can resume from where they left off. On WebGL this data is stored in browser’s IndexedDB.
This became interesting when we gave out a new build because the previous local state is not compatible with a new build, so the entire local state is empty and you start fresh.
Each build has a unique hash which separates the storage spaces on indexedDB because of which a new build starts with an empty state. The way this impacted us was everytime we gave a new build, the player would start their game from scratch and would have to go through FTUEs and then relink their profile with Facebook to retrieve their last playable account which led to bad UX.
The only way to solve this is that you need to be able to repair your local state if something goes wrong so you can avoid such problems.

### **Browsers support all languages but does your font too?**  
Our game is available in 14 languages but there were 5 that didn’t work on WebGL. Same font works on Android, IOS and Facebook Gameroom for all languages but not WebGL, why?
The answer is non trivial, it’s got to do with how unity works under the hood. Unity will first check for the unicode character in your font, if there’s a miss it will fallback to the default system fonts and look for the same unicode. The font that we currently use supports glyphs only for 8 languages out of the 14 that we support, but since our font is pretty close to the system default font we never realised that.
But why does it not fallback to system default on WebGL? Because your webapp runs in a restrictive environment and you don’t have access to the file system so you can’t use system default fonts on WebGL.
A font that would support all the languages we wanted would take somewhere close to 30MB and since we are already in a crunch of memory and bandwidth we chose to disable those 5 languages on WebGL.

### **Calling Custom Javascript with C# Adapter**  
You might encounter cases where you have to interact with certain web components which are not directly exposed to Unity. In such cases you can write your logic in javascript and call it using a C# adapter. 
In our game we are using a plugin UniWebView which is used to display web components in game but this plugin doesn’t support WebGL platform. So we thought to display the content in a new tab on the browser itself whenever required. So we wrote a simple function in javascript responsible for opening a webpage in a new tab. This custom logic written in JS is invoked by our C# scripts using a custom adapter again written in C#. Please follow the code example below. 
You just need to place your JS logic in a file with extension .jslib and place that under the Plugins subfolder inside the Assets folder.
```
mergeInto(LibraryManager.library, {

    OpenInNewTab: function (url) {
        window.open(Pointer_stringify(url));
    }
});   
```
You can call this JS logic from your C# Scripts using a C# adapter which would look something like this:  

```
#if UNITY_WEBGL
using System.Runtime.InteropServices;
 
namespace WebGL.Platform
{
    public class WebGLBrowserInterfaceUtils
    {
        [DllImport("__Internal")]
        public static extern void OpenInNewTab(string url);    
    }
}
#endif
```
You can now use the C# adapter normally like this:

```
using WebGL.Platform;
#if UNITY_WEBGL
      WebGLBrowserInterfaceUtils.OpenInNewTab(url);
#else
      Application.OpenURL(url);
#endif
```

### **Measuring and Debugging Runtime Memory**  
You can place the following javascript snippet in your game’s WebGL template which will print out the memory consumption onto your browser’s console every few seconds. This can be really helpful while debugging memory usage. You can extend this and build even more sophisticated use case of storing this information from all your production client users for analytical purposes

```
setInterval(function() {

  if (typeof TOTAL_MEMORY !== 'undefined') {
    try {
      var totalMem = TOTAL_MEMORY/1024.0/1024.0;
      var usedMem = (TOTAL_STACK + (STATICTOP - STATIC_BASE) + 
                    (DYNAMICTOP - DYNAMIC_BASE))/1024.0/1024.0;
      console.log('Memory stats - used: ' + Math.ceil(usedMem) + 'M' + 
                  ' free: ' + Math.floor(totalMem - usedMem) + 'M');
    } catch(e) {}
  }
}, 5000);
```

### Caching with Indexed DB

Your players need not download the entire client build everytime they access your game over the internet. Build data can be cached into IndexedDB the first time you download the client over the wire, so for subsequent accesses the client data can directly be picked from IndexedDB saving time and network bandwidth. Under the build player settings, set the Data Caching flag to true

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl6.png">
</p>

Or programmatically you can use the following:

```
PlayerSettings.WebGL.dataCaching = true;
```

### Using the right animation API on browser for better performance

There are 2 APIs browsers expose to animate - setTimeout and requestAnimationFrame. requestAnimationFrame API is considered superior to setTimeout since it automatically optimises and syncs the frame rate on browsers. setTimeout on the other hand assumes 60 frames per second and the user’s system might run on a different frame rate so your animation is already out of sync with the system.
Also setTimeout keeps running in background even if the tab is not active which may be unnecessary and may DOS your CPU which is not the case with requestAnimationFrame.
In order to use the superior API requestAnimationFrame for the smoothest animation set Application.targetFrameRate to a default value of -1.
You may want to throttle CPU in certain scenarios in your game, for which you can use Application.targetFrameRate API to do so. If it is not set to -1 then setTimeout will be used under the hood.

### Analysing Unity Build Reports

Unity prints out a descriptive build report on its console after a build is given out which is extremely helpful in analysing memory consumption of different components in game. 

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl7.png">
</p>

This comes in really handy when trying to optimise memory on the WebGL platform.

### Using WebAssembly

There are 2 linker targets supported by Unity - asm.js and webassembly. Webassembly is the way to go, asm.js is already deprecated in unity 2019. Wasm is faster, smaller and more memory-efficient than asm.js, which are all pain points of the Unity WebGL export. Wasm may not solve all the existing problems, but it certainly improves the platform in all areas.
One of the major advantages of using wasm is dynamic heap. It supports automatic heap resizing at runtime. You can leverage this if you are using Unity 2018.2+, but our game has long been on 2017.4 where dynamic heap is not supported by wasm. A trick that helped us achieve this on unity 2017.4 is by using the following emscripten argument along with the wasm linker:

```
PlayerSettings.WebGL.emscriptenArgs = “-s ALLOW_MEMORY_GROWTH=1”;
```

In order to use wasm instead of asm.js set the Linker Target to WebAssembly under the Build Player Settings:

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl8.png">
</p>

Or set it programmatically in your C# script using the following snippet:

```
PlayerSettings.WebGL.useWasm = true;
```

### WebGL Templates

The default HTML template that comes out of the box with Unity looks something like this:

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl9.png">
</p>

You may wanna spend some time modifying HTML CSS to make the canvas responsive and look production ready. 

### Publishing Platform

I think it is a no brainer, Facebook has been the number one option for developers to publish their games on the web. FB has a huge gaming audience and so discovery becomes easier on such platforms. The process of publishing your client on Facebook is quite straightforward. There are no reviews required and with a single click you can upgrade or rollback your clients immediately which is pretty awesome.
Another cool thing about hosting your game client on Facebook is that they don’t charge you extra for this. They manage hosting and their CDNs are pretty good and reliable.
You can also package your streaming assets and asset bundles with the client build itself so you can save bandwidth cost of S3/Akamai.

## Conclusion

We already see how web apps are replacing native apps these days on desktop as well as mobile. Every platform comes with its own set of challenges and we just explored a couple of them with WebGL.
 
With wasm you don’t have to specify unity heap size at build time, it automatically resizes at runtime (Unity 2018 +). In future Unity WebGL will also support multithreading using which we will be able to leverage multiple CPU cores all because of SharedArrayBuffers support on browsers. Though TBH, wasm already supports multithreading but due to some CPU exploits this feature is not stable on Unity and is currently under development. We can expect multithreading support on Unity WebGL by 2022.
 
But WebGL as a platform has immense potential and is constantly growing every day. We shall expect great improvements in the future iterations.

<p align="center">
  <img src="{{site.baseurl}}/assets/images/webgl99.gif">
</p>