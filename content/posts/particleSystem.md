---
title: 'Compute Shader particle system with Depth collission'
date: 2025-03-26T15:32:46+01:00
draft: false
hidden: false
externalURL: false
showDate: false
showModDate: false
showReadingTime: false
showTags: false
showPagination: true
invertPagination: true
showToC: true
openToC: false
showComments: false
showHeadingAnchors: true
---


| Type          | Description   |
| -----------   | -----------   |
| **Engine**    | Penugine      |
| **Timeline**  | ~8 weeks 50%  | 


{{< video src="/Portfolio/minSpeedEffect.mp4" autoplay="false" loop="true" width="800" height="450" >}}  
<!--more-->

## What this is
This is a demonstration and explanation of a particle system designed as a tool for designers and procedural artists to create captivating effects for our game projects. While the primary goal was to develop this tool, a significant focus was also placed on becoming comfortable with compute shaders and techniques suited for this task.

With compute shaders playing an increasingly important role in offloading CPU computation and enabling effects at a scale that would be impractical with traditional CPU processing, befriending them felt core to my skill set

## Implementation

### Shader loading and buffer creation

Start by loading our precompiled shaderobject.

```c
bool DX11::LoadComputeShader(const char* aPath, ID3D11ComputeShader** aComputeShader)
{
	HRESULT result;

	std::ifstream csFile;
	csFile.open(aPath, std::ios::binary);
	std::string data = {std::istreambuf_iterator<char>(csFile), std::istreambuf_iterator<char>()};
	result = DX11::Device->CreateComputeShader(data.data(), data.size(), nullptr, aComputeShader);
	if (FAILED(result))
	{
		return false;
	}
	csFile.close();
	return true;
}
```

Next the buffer is created with the data of the particle struct, Amount of particles, and inital data in myParticleData.
```c
result = CreateStructuredBuffer(DX11::Device, sizeof(Particle), NUM_PARTICLE, &myParticleData[0], myStructuredBuffer.GetAddressOf());
if (FAILED(result))
{
	return false;
}
```

The emmiter setup has particle data as the following.
```c
struct Particle
{
	Vector4f position = {0,0,0,0};
	float lifetime = 0.f;
	Vector3f velocity = {0,0,0};
};
```

## Mesh initialization

For the mesh based setup, the data is would be.
```c
struct Particle
{
	Vector4f position = {0,0,0,0};
	Vector4f startpositionModel1 = {0,0,0,0};
	Vector4f startpositionModel2 = {0,0,0,0};
	Vector4f velocity = {0,0,0,0};
	Vector4f wishVelocity = {1,1,1,1};
	Vector4f vColor	  = {0,0,0,0};
    Vector4<unsigned int> boolData = {0,0,0,0};
};
```
With an accompanying example

{{< video src="/Portfolio/meshInit.mp4" autoplay="false" loop="true" width="800" height="450" >}}  


The start position is the vertex position of the mesh. vColor is the vertex color if the model supports it, and the boolean data is represented by unsigned integers. This is because floats are 4 bytes in HLSL, whereas in C++, they are 1 byte. Preferably, this should be wrapped in a way that ensures conversion only happens when moving the data to the GPU, minimizing the possibility of user error. However, the only steps where this data is used are during the initial setup and in the HLSL code itself, so user error should not occur


The following video provides an example for multiple meshes

{{< video src="/Portfolio/multipleMeshes.mp4" autoplay="false" loop="true" width="800" height="450" >}}  

For the buffer we created, we create an unordered access view. This allows our compute shaders to read from and write to the buffer. We also create a shader resource view so that our non-compute shaders can read the data as intended. Our vertex shader will use this to output the position of each mesh representing a particle

```c
result = CreateBufferUAV(DX11::Device, myStructuredBuffer.Get(), myParticleUAV.GetAddressOf());
if (FAILED(result))
{
	return false;
}
result = CreateBufferSRV(DX11::Device, myStructuredBuffer.Get(), myParticleSRV.GetAddressOf());
if (FAILED(result))
{
	return false;
}
```

The dispatch and render passes are as following.
```c
RunComputeShader(myComputeShader.Get(), 0, nullptr, nullptr, nullptr, 0, myParticleUAV.Get(), NUM_PARTICLE / 10, 10, 1);

myPreprocessedFrame.SetAsActiveTarget(DX11::DepthBuffer);
myTentativeState.blendState = BlendState::AlphaBlend;
UpdateGpuState();
DX11::Context->VSSetShaderResources(0, 1, myParticleSRV.GetAddressOf());
DX11::Context->PSSetConstantBuffers(11, 1, myParticleColorBuffer.GetAddressOf());
myParticleShader->Render(t);
```

This is all that needs to be done on the CPU. myParticleShader holds the meshes for the particles themselves. In this case, they are represented by a diamond to provide some volume while keeping the geometry minimal.


## Shaders

The following is a simple emitter that uses pseudo-random algorithms seeded with the particle ID. This will create a repeating pattern, however, given the number of particles, this is negligible.
```hlsl
[numthreads(1, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID)
{
    BufferInOut[DTid.x].lifetime.x += Timings.y;
    BufferInOut[DTid.x].velocity.y -= 0.00009f;
    if (BufferInOut[DTid.x].lifetime.x > 33.f)
    {
        BufferInOut[DTid.x].lifetime.x = 0.f;
        BufferInOut[DTid.x].velocity.xyz = up.xyz;
        BufferInOut[DTid.x].velocity.xyz += forward.xyz * (rand_xorshift(DTid.x) * (2.0 / 4294967296.0) - 1.f);
        BufferInOut[DTid.x].velocity.xyz += right.xyz * (rand_lcg(DTid.x) * (2.0 / 4294967296.0) - 1.f);
        BufferInOut[DTid.x].position.xyz = forcePosition;
    }
    
    BufferInOut[DTid.x].position.xyz += BufferInOut[DTid.x].velocity.xyz * Timings.y * 200.f;
}
```

This produces the following effect.
{{< video src="/Portfolio/emitterEx.mp4" autoplay="false" loop="true" width="800" height="450" >}}  


This effect uses the following vertex shader, with ObjectToWorld as an extra transform to rotate and scale the particles if so desired.
```hlsl
ParticlePSInput main(ParticleVSInput input)
{
    ParticlePSInput output;  
    
    float4 particlePos = Buffer0[input.id].position;
    
    
    float4x4 transform = float4x4(
    1, 0, 0, Buffer0[input.id].position.x,
    0, 1, 0, Buffer0[input.id].position.y,
    0, 0, 1, Buffer0[input.id].position.z,
    0, 0, 0, 1);
    float4x4 toNewObject = mul(transform, ObjectToWorld);
    float4 vertexWorldPos = mul(transform, input.position);
    float4 vertexViewPos = mul(WorldToCamera, vertexWorldPos);

    output.worldPosition = vertexWorldPos;
    output.position = mul(CameraToProjection, vertexViewPos);
    output.vertexColor0 = Buffer0[input.id].vColor;
    return output;
}
```
## Depth collission
As observed above, the particles simply fall through the ground, making them appear out of place in the 3D scene. This issue is resolved using depth collision.

To achieve this, we need to project the particles onto screen space to sample the depth and normal of the occupying pixels. The normal will be used upon collision for impact velocity calculations.

In the graphics pipeline I designed for our engine, the necessary data is stored in the geometry buffers. We bind these before the compute shader pass
```c
myGbuffer.CSSetAsResourceOnSlot(GBuffer::GBufferTexture::ScreenPos, 1);
myGbuffer.CSSetAsResourceOnSlot(GBuffer::GBufferTexture::Normal, 2);
```

In the compute shader, we convert the world position into the necessary coordinate spaces. We use view space for Z-depth from our camera and projected space for screen-space coordinates to sample the pixel depth and normals

```hlsl
    float4x4 transform = float4x4(
1, 0, 0, BufferInOut[DTid.x].position.x,
0, 1, 0, BufferInOut[DTid.x].position.y,
0, 0, 1, BufferInOut[DTid.x].position.z,
0, 0, 0, 1);

float4 vertexWorldPos = mul(transform, float4(0, 0, 0, 1));
float4 vertexViewPos = mul(WorldToCamera, vertexWorldPos);
float4 projected = mul(CameraToProjection, vertexViewPos);
```

We then convert the projected values into Direct3D11 projection space using the following method.
```hlsl
projected.xyz /= projected.w;
projected.xy = (projected.xy + 1) * 0.5f;
projected.y = 1 - projected.y;
```

Next, we sample the projected pixel and compare it to the particle's depth. If the particle is within a certain range of the projected depth, we apply a smear collision and add a bit of bounce, depending on the velocity and the angle at which the collision occurred.

```hlsl
float depth = DepthTexture.SampleLevel(defaultSampler, projected.xy, 0).z;
if (vertexViewPos.z < depth + radius && vertexViewPos.z > depth - radius)
{
    float3 normal = NormalTexture.SampleLevel(defaultSampler, projected.xy, 1).xyz * 2 - 1;
    if (length(BufferInOut[DTid.x].velocity.xyz) < 0)
    {
        return;
    }
    BufferInOut[DTid.x].velocity.xyz = BufferInOut[DTid.x].velocity.xyz - normal * dot(BufferInOut[DTid.x].velocity.xyz, normal) + normal * max(0, -dot(BufferInOut[DTid.x].velocity.xyz, normal) * length(BufferInOut[DTid.x].velocity.xyz) - 2);
}
```


This produces the following effect.
{{< video src="/Portfolio/depthGeneratorNew.mp4" autoplay="false" loop="true" width="800" height="450" >}}  



A clear flaw with this technique is that if the object is not visible, it does not contribute to the depth buffer, and the particles do not collide. The following video demonstrates this issue.

{{< video src="/Portfolio/needVisuals.mp4" autoplay="false" loop="true" width="800" height="450" >}}  



## Areas of improvement

Stretch goals I personally had included implementing motion blur for the particles to simulate higher velocities, making the effect feel more dynamic.

A system like this is very flexible. New features can always be added to enable different implementations. Making the system more modular for in-game use and providing more customization options for the effects produced are also high on the priority list.