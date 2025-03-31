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


{{< video src="minSpeedEffect.mp4" autoplay="false" loop="true" width="800" height="450" >}}  
<!--more-->

# What this is
This is a demonstration and explanation of a particle system designed as a tool for designers and procedural artists to create captivating effects for our game projects. While the primary goal was to develop this tool, a significant focus was also placed on becoming comfortable with compute shaders and techniques suited for this task.

With compute shaders playing an increasingly important role in offloading CPU computation and enabling effects at a scale that would be impractical with traditional CPU processing, befriending them felt core to my skill set

# Implementation

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

Next the buffer is created, with the data of the particle struct. Amount of particles, and inital data in myParticleData.
```c
result = CreateStructuredBuffer(DX11::Device, sizeof(Particle), NUM_PARTICLE, &myParticleData[0], myStructuredBuffer.GetAddressOf());
if (FAILED(result))
{
	return false;
}
```

The emmiter setup has particle data like the following.
```c
struct Particle
{
	Vector4f position = {0,0,0,0};
	float lifetime = 0.f;
	Vector3f velocity = {0,0,0};
};
```

For the mesh based setup, the data is as folliwng.
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
With accompanning example

{{< video src="meshInit.mp4" autoplay="false" loop="true" width="800" height="450" >}}  


Where the start positions is the vertex position of the mesh. vColor is the vertex color if the model supports it, and bool data is. The bools are represented by unsigned integers because floats are 4 bytes in hlsl while in c++ they are 1 byte. Prefeably this should be wrapped in a way where this is only done when moving the data to the GPU to remove the possibility of user error. However the only steps where this data is used is durring the inital setup, and in the hlsl code itself. So user error ought not to occur.

The following video provides an example of multiple meshes

{{< video src="multipleMeshes.mp4" autoplay="false" loop="true" width="800" height="450" >}}  

For the buffer we created, we create a unordered access view. This is used to read and write to the buffer from our compute shaders. We also create a shader resource view, so our non compute shaders can read the data in their intended way. This will be done by our vertex shader to output the posision of each mesh representing a particle.

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

The dispatch and render passes are the following
```c
myGbuffer.CSSetAsResourceOnSlot(GBuffer::GBufferTexture::ScreenPos, 1);
myGbuffer.CSSetAsResourceOnSlot(GBuffer::GBufferTexture::Normal, 2);
RunComputeShader(myComputeShader.Get(), 0, nullptr, nullptr, nullptr, 0, myParticleUAV.Get(), NUM_PARTICLE / 10, 10, 1);

myPreprocessedFrame.SetAsActiveTarget(DX11::DepthBuffer);
myTentativeState.blendState = BlendState::AlphaBlend;
UpdateGpuState();
DX11::Context->VSSetShaderResources(0, 1, myParticleSRV.GetAddressOf());
DX11::Context->PSSetConstantBuffers(11, 1, myParticleColorBuffer.GetAddressOf());
myParticleShader->Render(t);
```

This is all that has to be done on the cpu, myParticleShader holds the meshes for the particles themselves, in this case it is just represented by a diamond to give some volume while also keeping the geometry minimal.


The following is a simple emmiter using pseudo random algorithms seeding with particle id (this will make a repeating pattern, however, with the amount of partilces this is negligible)
```c
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
{{< video src="emitterEx.mp4" autoplay="false" loop="true" width="800" height="450" >}}  

```c
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

{{< video src="depthBufferExample.mp4" autoplay="false" loop="true" width="800" height="450" >}}  



{{< video src="needVisuals.mp4" autoplay="false" loop="true" width="800" height="450" >}}  



# Areas of improvement