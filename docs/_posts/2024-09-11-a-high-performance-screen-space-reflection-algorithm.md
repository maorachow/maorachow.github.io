---
layout: post
title:  "A High Performance Screen Space Reflection Algorithm"
date:   2024-09-11 15:25:26 +0800
categories: 3D Graphics

---

Unity team won't accept my asset until I show them what aspect of programming I am good at. So I post this article to show what kind of skill I have. Back to the topic.

## A High Performance Screen Space Reflection Algorithm: Implemented In Unity

### Intro to SSR

Screen space reflections, also known as SSR, is a screen-space effect used to simulate reflections in 3D space on any surface. Although it can only reflect surfaces in screen space, it still greatly increases the rendering quality of a game because it has high precision and can fit in complex scenes more accurately than IBL or planear reflections.    

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Sponza_without_SSR.png?raw=true)

Sponza scene without SSR

![Sponza with SSR](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Sponza%20with%20SSR.png?raw=true)

### Performance issues

High quality often means high performance impact, that's true in our topic: Screen space reflections. Code below is a straight-ahead implemention of the SSR algorithm: get the world space origin and the reflection ray direction of the shading point, then march along the direction step by step, until an intersection is found. That's the basic "World Space Raymarching" method.

```hlsl
            float4 DirectSSR(Varyings input):SV_TARGET{
              float rawDepth1=SampleSceneDepth(input.texcoord); 
              float linearDepth = LinearEyeDepth(rawDepth1, _ZBufferParams);

              float3 vnormal = (SampleSceneNormals(input.texcoord).xyz);

              float3 rayOriginWorld = ReconstructViewPos(input.texcoord,linearDepth)+ProjectionParams2.yzw; 

              float3 vDir=normalize(rayOriginWorld - ProjectionParams2.yzw);
              float3 rDir = reflect(vDir, normalize(vnormal));

              float3 marchPos=rayOriginWorld+vnormal*0.3*(linearDepth/100.0);//A small bias to prevent self-reflections
              float strideLen=0.05;//stride length
              UNITY_LOOP
              for(int i=0;i<MaxIterations;i++){
                marchPos+=strideLen*rDir;//Get a new march position
                float3 viewPos=mul(UNITY_MATRIX_V,float4(marchPos,1)).xyz;//In unity matrix mulplication, matrix should be on the left

                float4 projectionPos=mul(UNITY_MATRIX_P,float4(viewPos,1)).xyzw;//Transform the point to the texture space
                projectionPos.xyz/=projectionPos.w;
                projectionPos.xy=  projectionPos.xy*0.5+0.5;
                #if UNITY_UV_STARTS_AT_TOP
                float2 uv=float2(projectionPos.x,1-projectionPos.y);
                #else
                float2 uv=float2(projectionPos.x,projectionPos.y);
                #endif
                float testDepth=projectionPos.z;//Compare the result depth value by the testing-point texture space transformation
                float sampleDepth=SampleSceneDepth(uv);//With the depth sampled from the scene using UV by the testing-point texture space transformation

                float linearTestDepth=LinearEyeDepth(testDepth, _ZBufferParams);//Use linear eye depth for intersection testing
                float linearSampleDepth=LinearEyeDepth(sampleDepth, _ZBufferParams);

                if(uv.x<0||uv.x>1||uv.y<0||uv.y>1){
                break;//Terminate testing points that are out of the screen space
                }
                #define DEPTH_TESTING_THERESHOLD 0.2
                if(linearTestDepth>linearSampleDepth&&abs(linearSampleDepth-linearTestDepth)<DEPTH_TESTING_THERESHOLD){//If the testing point is below the surface and not too much below
                if(linearTestDepth<_ProjectionParams.y||linearTestDepth>_ProjectionParams.z*0.9){
                break;//Terminate intersections that are out of range
                }
                 return SAMPLE_TEXTURE2D_X(_BlitTexture,sampler_point_clamp,uv);//Simply return scene color of the intersection point.
                }

              }
              return float4(0,0,0,0);//Not intersected 
            }
```

This is a basic "world space raymarching" version of SSR. Takes very short time to code, right? Let's see the performance and quality of this method.

![Bad FPS](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Bad%20FPS.png?raw=true "Bad FPS")

Rendered in 4K resolution, 1024 iterations, marching stride length: 0.05 meters, 17.5FPS, no reflection at further distance.

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Bad%20FPS2.png?raw=true)

Rendered in 4K resolution, 1024 iterations, marching stride length: 0.25 meters, 22FPS, stripped artifacts at further distance.

The result is not very acceptable, due to 2 critical issues:

#### 1.Over-sampling

As the image one shows, there are less reflections at further distance. That's because our current method marches the ray in the world space, because we need to project this "world space position" into the screen space to compare with the depth value stored in camera depth texture, the delta of screen space testing UV in each iteration will not be as much as the world space position. A bad case is like that:  we have done 1000 steps of raymarching in a pixel, but it only moves the testing point a few pixels further, causing a waste of time.

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Over-sampling.jpg?raw=true)

#### 2.Useless testing in the air

Our current method just moves the testing point by a constant distance, if the surface is very far away from the camera, the testing point may not reach the surface after finishing all the steps, so you cannot have distant reflections with low testing step count and you cannot have high performance with high testing step count. Instead of having a fixed distance every step, can we use a dynamic step length and skip empty spaces to make the algorithm more effective?

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Constant%20Ray%20Step%20Length.png?raw=true)

### An universal solution

Is there a "super algorithm" that can solve both two major issues? The answer is **YES**. Let's solve these issues.

#### Marching In The Texture Space

We know that the perspective projection matrix transforms primitives from view space to homogeneous space. The perspective projection shrinks all the objects in the view frustum into a "Homogeneous clip Space" which maybe different depending on your graphics API, we use a perspective divide to convert HCS into Normalized Device Coordinates which is similar to a cube. In this "cube" all the coordinate values are between (X=-1,Y=-1,Z=-1) and (X=1,Y=1,Z=1).(On directX platforms, between (X=-1,Y=-1,Z=0) and (X=1,Y=1,Z=1))  then we remap this "NDC Cube" to fit in a 2D texture and use the Z value stored in it for depth testing, this texture is the final image presented on the screen. Here I use images on www.opengl-tutorial.org to visualize this progress.

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Before%20Perspective.png?raw=true) 

![After Perspective](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Homogeneous%20Space.png?raw=true "After Perspective")

Since all visible objects are transformed into this "NDC Cube", we can raymarch in NDC space and use pixels as the unit of ray step length, so every pixel on the ray path will be covered and a single ray will not sample a pixel multiple times. In order to fit in the texture and make sampling easier, we do the raymarch in "Texture space". "Texture space" is just a remap of the "NDC Cube". Issue 1 solved!

![Texture Space](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Texture%20Space.png?raw=true "Texture Space")

Now we can upgrade our method from "World Space Raymarching" to "Texture Space Raymarching". Here's the code.

```hlsl
            float4 TextureSpaceSSR(Varyings input):SV_TARGET{
                float rawDepth1=SAMPLE_TEXTURE2D_X_LOD(HiZBufferTexture,sampler_point_clamp,input.texcoord,0);//"HiZBufferTexture" is just used as regular depth buffers here 
                float linearDepth = LinearEyeDepth(rawDepth1, _ZBufferParams);
                if(linearDepth>_ProjectionParams.z*0.9){
                return float4(0,0,0,0);
                }
                float3 vnormal = (SampleSceneNormals(input.texcoord).xyz);

                float3 rayOriginWorld = ReconstructViewPos(input.texcoord,linearDepth)+ProjectionParams2.yzw;

                float3 vDir=normalize(rayOriginWorld - ProjectionParams2.yzw);
                float3 rDir = reflect(vDir, normalize(vnormal));//Get world space position and reflecting direction of current shading point first

                float maxDist=500.0; 

                float3 rayOriginInVS=mul(UNITY_MATRIX_V,float4(rayOriginWorld,1)).xyz;

                float4 rayOriginHS=mul(UNITY_MATRIX_P,float4(rayOriginInVS,1)).xyzw;//Homogeneous clip space ray origin

                float3 rDirInVS=mul(UNITY_MATRIX_V,float4(rDir,0)).xyz;

                float end = rayOriginInVS.z + rDirInVS.z * maxDist;
                if (end > - _ProjectionParams.y)
                {
                   maxDist = (-_ProjectionParams.y - rayOriginInVS.z) / rDirInVS.z; //Prevent the endpoint from going to the back side of camera
                }

                float3 rayEndInVS=rayOriginInVS+rDirInVS*maxDist;//First transform the ray origin and direction into view space to get the endpoint of the ray in view space


                float4 rayEndHS=mul(UNITY_MATRIX_P,float4(rayEndInVS.xyz,1));
                rayEndHS.xyz/=rayEndHS.w;
                rayOriginHS.xyz/=rayOriginHS.w;//Perspective devide 

                float3 rayOriginTS=float3(rayOriginHS.x*0.5+0.5,(rayOriginHS.y*0.5+0.5),rayOriginHS.z);
                float3 rayEndTS=float3(rayEndHS.x*0.5+0.5,(rayEndHS.y*0.5+0.5),rayEndHS.z);

                #if UNITY_UV_STARTS_AT_TOP
                rayOriginTS.y=1-rayOriginTS.y;
                rayEndTS.y=1-rayEndTS.y;
                #endif

                #if UNITY_REVERSED_Z
                //Unity uses reversed Z method to store depth data of the scene for better precision in far distance so
                //the depth value is between 1 and 0. I tried to manually re-reverse it into [0,1] but ran into aliasing
                //probably due to precision lost, so I commented code below out.
                //
                //  rayOriginTS.z=1.0-rayOriginTS.z;
                //  rayEndTS.z=1.0-rayEndTS.z;
                #endif

                float3 rDirInTS=normalize(rayEndTS-rayOriginTS);//Reflection ray direction in Texture Space
                float outMaxDistance = rDirInTS.x >= 0 ? (1 - rayOriginTS.x) / rDirInTS.x : -rayOriginTS.x / rDirInTS.x;
                outMaxDistance = min(outMaxDistance, rDirInTS.y < 0 ? (-rayOriginTS.y / rDirInTS.y) : ((1 - rayOriginTS.y) / rDirInTS.y));
                outMaxDistance = min(outMaxDistance, rDirInTS.z < 0 ? (-rayOriginTS.z / rDirInTS.z) : ((1 - rayOriginTS.z) / rDirInTS.z));//Use a "max distance" to clamp the textue space reflection endpoint



               float3 reflectionEndTS=rayOriginTS + rDirInTS * outMaxDistance;//to make sure it won't go out the "Texture Space" box
               float3 dp = reflectionEndTS.xyz - rayOriginTS.xyz;//Texture Space delta between the ray origin and endpoint
               float2 originScreenPos = float2(rayOriginTS.xy *SSRSourceSize.xy);//The pixel position of the ray origin and endpoint. SSRSourceSize.xy stands for the width and height of screen.
               float2 endPosScreenPos = float2(reflectionEndTS.xy *SSRSourceSize.xy);
               float2 pixelDelta=endPosScreenPos - originScreenPos;//Pixel delta between the ray origin and endpoint
               float max_dist = max(abs(pixelDelta.y), abs(pixelDelta.x));//Get max value between two components of pixelDelta
               dp /= max_dist;//Divide dp by max_dist to get raymarching stride length so we can make sure every pixel along the ray route will be covered.

                float4 marchPosInTS = float4(rayOriginTS.xyz+dp, 0);
                float4 rayDirInTS = float4(dp.xyz, 0);
                float4 rayStartPos = marchPosInTS;
                bool isIntersected=false;
                UNITY_LOOP
                    for(int i = 0; i<MaxIterations;i ++)
                    {
                    float rawDepth = SAMPLE_TEXTURE2D_X_LOD(HiZBufferTexture,sampler_point_clamp,marchPosInTS.xy,0);
                    float testRawDepth=marchPosInTS.z;
                    #if UNITY_REVERSED_Z
              //      testRawDepth=1.0-testRawDepth;
                    #endif
                    float sampleLinearDepth=LinearEyeDepth(rawDepth,_ZBufferParams);//Use linear depth to check intersection
                    float testLinearDepth=LinearEyeDepth(testRawDepth,_ZBufferParams);

                    float thickness = abs(testLinearDepth - sampleLinearDepth);


                    if(marchPosInTS.x<0||marchPosInTS.y<0||marchPosInTS.x>1||marchPosInTS.y>1 )
                    {
                         isIntersected=false;
                        break;
                    }
                    #define MAX_THICKNESS_DIFFERENCE_TO_HIT 0.6
                    if(testLinearDepth>=sampleLinearDepth&&thickness<MAX_THICKNESS_DIFFERENCE_TO_HIT)//We use a constant thickness to decide has the test point intersected a surface or it just go below the surface,

                    {
                    if(testLinearDepth>_ProjectionParams.z*0.9||sampleLinearDepth>_ProjectionParams.z*0.9){
                    isIntersected=false;
                        break;
                    }
                         isIntersected=true;
                        break;
                    }

                        marchPosInTS += rayDirInTS;
                    }
                    if(isIntersected==true){
                    float2 uv=marchPosInTS.xy;
                    return SAMPLE_TEXTURE2D_X(_BlitTexture,sampler_point_clamp,uv);

                    }else{
                    return float4(0,0,0,0);
                    }
            }
```

In code above we get the ray origin and endpoint in texture space and march from the origin to the end **one pixel every step**. "Texture Space Raymarching" can solve the "Over-sampling" issue said above.

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Texture%20Space%20SSR1.png?raw=true)

Rendered in 4K resolution, 384 raymarching iterations. Notice that the reflected image length is not influenced by the distance between the shading point and the camera. That's especially good when you need distant reflections.

Note that we are stepping in the "Texture Space" with a **same** step distance, but in world space the distance between testing points may change because of perspective projection.

<img title="" src="https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Texture%20Space%20Not%20Linear.png?raw=true" alt="@" data-align="center">

#### Hierarchical-Z Tracing

Now our "Texture Space Raymarching" method still has a big problem: performance. We are moving our testing point towards the end one pixel per step. If we want to make a ray fully travel through the screen, we need thousands of marching step and that is too much for realtime rendering. Hierarchical-Z tracing method is the key to reduce marching step count and boost rendering speed. 

Hierarchical-Z buffer is a mipmap chain that contains from detailed to undetailed scene  depth data. The first mipmap level of it is just a copy of the camera depth buffer, each pixel in the second or above mipmap level takes the **minimum or maximum** depth value from 4 neighbouring pixels in previous mipmap level and store the value as its data. Figure below represents how this mipmap generation works:

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Hi-Z%20mipmap%20chain.png?raw=true)

Building such kind of hierarchical structure can help the raymarching algorithm effectively skip empty spaces in scene by using a larger step length and do depth comparisation in higher mipmap levels. In order not to produce a wrong intersection result, we construct the Hi-Z buffer by taking the most "near from the camera" depth value from 4 neighbouring pixels in previous mipmap level.

<img title="" src="https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Hi-Z%20Mipmap%20Chain1.png?raw=true" alt="@" data-align="center" width="481">

The general idea of tracing with hierarchical-Z buffer is like the pseudo code below:

```hlsl
int level=0

point ray=texture space ray origin
direction dir=texture space ray direction

finalIntersectionPoint result
while(iteration less than max allowed){

    gridCell currentCellOfRay=Get which grid cell in current level the ray point in
    float depthInCurrentCell=Sample Hi-Z Buffer in current level
    point testRayPoint=Intersect the texture space ray with the plane z=depthInCurrentCell
    gridCell CellOfTestRayPoint=Get which grid cell in current level the testRayPoint in

    if(CellOfTestRayPoint and currentCellOfRay is not the same cell){
    ray=the intersection between the texture space ray and the grid border in current level
    not intersected, ascend a level
    }
    if(CellOfTestRayPoint and currentCellOfRay is the same cell){
    if(current level is stop level){
    result=ray
    break
    }
      ray= testRayPoint
     intersected, decend a level
    }

}
color=Get reflection of the result
```

And the the ray will act like the image below:

<img title="" src="https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Hierarchical-Z%20Tracing.png?raw=true" alt="@" data-align="center">

Note that in the pseudo I write "texure space", that's because the Hi-Z tracing will happen in "Texture Space" said in the paragraph Marching In The Texture Space. Hi-Z tracing is based on texture space raymarching method, that means it can solve both two major issues I listed.

Because of my poor english speaking, I cannot explain the details of how to fully implement the Hi-Z tracing algorithm clearly, so I just put sugulee's blog about Hi-Z Tracing method here.[Screen Space Reflections : Implementation and optimization – Part 2 : HI-Z Tracing Method – Sugu Lee (wordpress.com)](https://sugulee.wordpress.com/2021/01/19/screen-space-reflections-implementation-and-optimization-part-2-hi-z-tracing-method/)

This blog is a clear and thorough explanation on what the Hi-Z tracing method works and how to turn this method from theory to code. It's also where I learned the Hi-Z Tracing method.

### Result and conclusion

Time to show the power of Hierarchical-Z Tracing, our new high-performance method of Screen Space Reflection.

All screenshots are rendered in 4K resolution, 100 max iterations.

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Hi-Z%20Tracing.png?raw=true)

Hi-Z Tracing, 160-190 FPS, very far reflections

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Texture%20Space%20Raymarching.png?raw=true)

Texture Space Raymarching, 200-220FPS, near reflections

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/World%20Space%20Raymarching.png?raw=true)

World Space Raymarching, 160-180FPS, very near reflections

![@](https://github.com/maorachow/maorachow.github.io/blob/main/docs/images/20240911/Texture%20Space%20Raymarching1.png?raw=true)

Texture Space Raymarching,1024 iterations, 50-60FPS, far reflections

As you can see, the Hi-Z Tracing method has the longest tracing distance with only 100 step iterations and has no noticeable artifacts. The Hi-Z Tracing method is truely a fantastic technique. It can also be used on effects related to screen space raymarching, such as Screen Space Contact Shadows, Screen Space Indirect Diffuse.

While implementing this method in Unity, I met a small issue: reversed Z-buffer. Some lines of code in sugulee's blog compares Z component of the texture space coordinates to decide is the ray going backwards or has the ray intersected a depth plane, but in Unity the depth value in depth buffer is reversed on DirectX platforms, causing these logic to break. The solution of this issue is not complex, we can just use different codes on different platforms. For example: 

Original code:

```hlsl
  bool isBackwardRay = vReflDirInTS.z<0;
    float rayDir = isBackwardRay ? -1 : 1;
```

Fixed code:

```hlsl
  #if UNITY_REVERSED_Z
     bool isBackwardRay = reflectDirTextureSpace.z> 0;
     float rayDir = isBackwardRay ? 1 : -1;
  #else
   bool isBackwardRay = reflectDirTextureSpace.z< 0;
     float rayDir = isBackwardRay ? -1 : 1;

  #endif
```

That's almost all of the content, Thanks sugulee for giving out a good explaination of the Hi-Z tracing method, that really helps! This is the first time I post a blog, hope my article helps!
