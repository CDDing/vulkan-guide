---
layout: default
title: Descriptor Abstractions
parent: "4. Textures and Engine Architecture"
nav_order: 2
---

Now that we are going to be growing the engine abstractions to support textures and considerably increase the complexity, we are going to need better abstractions for the descriptor sets.

In chapter 2, we already created 2 classes, the Descriptor Allocator and Descriptor Layout Builder. With the descriptor Allocator we have a basic way of abstracting a single VkDescriptorPool to allocate descriptors, and the LayoutBuilder abstracts creating Descriptor Set Layouts.

# Descriptor Allocator 2
We are going to create a new version of the Descriptor Allocator, `DescriptorAllocatorGrowable`. The one we created before will just crash when the pool runs out of space. This is fine for some cases where we know the amount of descriptors ahead of time, but it wont work when we need to load meshes from arbitrary files and cant know ahead of time how many descriptors we will need. This new class will perform almost exactly the same, except instead of handling a single pool, it handles a bunch of them. Whenever a pool fails to allocate, we create a new one. When this allocator gets cleared, it clears all of its pools. This way we can use 1 descriptor allocator and it will just grow as we need to.

This is the implementation we will have in the header at vk_descriptors.h 

^code descriptor_allocator_grow shared/vk_descriptors.h 

The public interface is the same as in the other descriptor allocator. What has changed is that now we need to store the array of pool size ratios (for when we reallocate the pools), how many sets we allocate per pool, and 2 arrays. `fullPools` contains the pools we know we cant allocate from anymore, and `readyPools` contains the pools that can still be used, or the freshly created ones. 

The allocation logic will first grab a pool from readyPools, and try to allocate from it. If it succeeds, it will add the pool back into the readyPools array. If it fails, it will put the pool on the fullPools array, and try to get another pool to retry. The `get_pool` function will pick up a pool from readyPools, or create a new one.

Lets write the get_pool and create_pool functions

^code growpool_1 shared/vk_descriptors.cpp 

On get_pools, when we create a new pool, we increase the setsPerPool, to mimic something like a std::vector resize. Still, we will limit the max amount of sets per pool to 4092 to avoid it growing too much. This max limit can be modified if you find it works better in your use cases.

An important detail on this function is that we are removing the pool from the readyPools array when grabbing it. This is so then we can add it back into that array or the other one once a descriptor is allocated.

On the create_pool function, its the same we had in the other descriptor allocator.

Lets create the other functions we need, init(), clear_pools(), and destroy_pools()

^code growpool_2 shared/vk_descriptors.cpp 

the init function just allocates the first descriptor pool, and adds it to the readyPools array.

clearing the pools means going through all pools, and coping the fullPools array into the readyPools array. 

destroying loops over both lists and destroys everything to clear the entire allocator.

Last is the new allocation function.

^code growpool_3 shared/vk_descriptors.cpp 

We first grab a pool, then allocate from it, and if the allocation failed, we add it into the fullPools array (as we know this pool is filled) and then try again. If the second time fails too stuff is completely broken so it just asserts and crashes. Once we have allocated with a pool, we add it back into the readyPools array.

# Descriptor Writer
When we needed to create a descriptor set for our compute shader, we did the vulkan `vkUpdateDescriptorSets` the manual way, but this is really annoying to deal with. So we are going to abstract that too. In our writer, we are going to have a `write_image` and `write_buffer` functions to bind the data. Lets look at the struct declaration, also on the vk_descriptors.h file.

^code writer shared/vk_descriptors.h 

We are doing some memory tricks with the use of std::deque here. std::deque is guaranteed to keep pointers to elements valid, so we can take advantage of that mechanic when we add new `VkWriteDescriptorSet` into the writes array. 

Lets look at the definition of `VkWriteDescriptorSet`
```cpp
typedef struct VkWriteDescriptorSet {
    VkStructureType                  sType;
    const void*                      pNext;
    VkDescriptorSet                  dstSet;
    uint32_t                         dstBinding;
    uint32_t                         dstArrayElement;
    uint32_t                         descriptorCount;
    VkDescriptorType                 descriptorType;
    const VkDescriptorImageInfo*     pImageInfo;
    const VkDescriptorBufferInfo*    pBufferInfo;
    const VkBufferView*              pTexelBufferView;
} VkWriteDescriptorSet;
```

we have target set, target binding element, and the actual buffer or image is done by pointer. We need to keep the information on the VkDescriptorBufferInfo and others in a way that the pointers are stable, or a way to fix up those pointers when making the final WriteDescriptorSet array.

Lets look at what the write_buffer function does.

^code write_buffer shared/vk_descriptors.cpp 

We have to fill a VkDescriptorBufferInfo first, with the buffer itself, and then an offset and range (size) for it. 

Then, we have to setup the write itself. Its only 1 descriptor, at the given binding slot, with the correct type, and a pointer to the VkDescriptorBufferInfo. We have created the info by doing emplace_back on the std::deque, so its fine to take a pointer to it.

The descriptor types that are allowed for a buffer are these. 

```
VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER
VK_DESCRIPTOR_TYPE_STORAGE_BUFFER
VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC
VK_DESCRIPTOR_TYPE_STORAGE_BUFFER_DYNAMIC
```

We already explained those types of buffers in the last chapter. When we want to bind one or the other type into a shader, we set the correct type here. Remember that it needs to match the usage when allocating the VkBuffer

For images, this is the other function.
^code write_image shared/vk_descriptors.cpp 

Very similar to the buffer one, but we have a different Info type, using a `VkDescriptorImageInfo` instead. For that one, we need to give it a sampler, a image view, and what layout the image uses. The layout is going to be almost always either `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`, the best layout to use for accessing textures in the shaders, or `VK_IMAGE_LAYOUT_GENERAL` when we are using them from compute shaders and writing them.

The 3 parameters in the ImageInfo can be optional, depending on the specific VkDescriptorType.

* `VK_DESCRIPTOR_TYPE_SAMPLER` is JUST the sampler, so it does not need ImageView or layout to be set. 
* `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE` doesnt need the sampler set because its going to be accessed with different samplers within the shader, this descriptor type is just a pointer to the image. 
* `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` needs everything set, as it holds the information for both the sampler, and the image it samples. This is a useful type because it means we only need 1 descriptor binding to access the texture.
* `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE` was used back in chapter 2, it does not need sampler, and its used to allow compute shaders to directly access pixel data.

In both the write_image and write_buffer functions, we are being overly generic. This is done for simplicity, but if you want, you can add new ones like `write_sampler()` where it has `VK_DESCRIPTOR_TYPE_SAMPLER` and sets imageview and layout to null, and other similar abstractions.

With these done, we can perform the write itself.


^code writer_end shared/vk_descriptors.cpp 

The clear() function resets everything. The update_set function takes a device and a descriptor set, connects that set to the array of writes, and then calls `vkUpdateDescriptorSets` to write the descriptor set to its new bindings.

Lets look at how this abstraction can be used to replace code we had before in the `init_descriptors` function


before: 
```cpp
VkDescriptorImageInfo imgInfo{};
imgInfo.imageLayout = VK_IMAGE_LAYOUT_GENERAL;
imgInfo.imageView = _drawImage.imageView;

VkWriteDescriptorSet drawImageWrite = {};
drawImageWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
drawImageWrite.pNext = nullptr;

drawImageWrite.dstBinding = 0;
drawImageWrite.dstSet = _drawImageDescriptors;
drawImageWrite.descriptorCount = 1;
drawImageWrite.descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_IMAGE;
drawImageWrite.pImageInfo = &imgInfo;

vkUpdateDescriptorSets(_device, 1, &drawImageWrite, 0, nullptr);
```

after:
```cpp
DescriptorWriter writer;
writer.write_image(0, _drawImage.imageView, VK_NULL_HANDLE, VK_IMAGE_LAYOUT_GENERAL, VK_DESCRIPTOR_TYPE_STORAGE_IMAGE);

writer.update_set(_device,_drawImageDescriptors);
```

This abstraction will prove much more useful when we have more complex descriptor sets, specially in combination with the allocator and the layout builder. 

# Dynamic Descriptor Allocation
Lets start using the abstraction by using it to create a global scene data descriptor every frame. This is the descriptor set that all of our draws will use. It will contain the camera matrices so that we can do 3d rendering.

To allocate descriptor sets at runtime, we will hold one descriptor allocator in our FrameData structure. This way it will work like with the deletion queue, where we flush the resources and delete things as we begin the rendering of that frame. Resetting the whole descriptor pool at once is a lot faster than trying to keep track of individual descriptor set resource lifetimes.

We add it into FrameData struct

```cpp
struct FrameData {
	VkSemaphore _swapchainSemaphore, _renderSemaphore;
	VkFence _renderFence;

	VkCommandPool _commandPool;
	VkCommandBuffer _mainCommandBuffer;

	DeletionQueue _deletionQueue;
	DescriptorAllocatorGrowable _frameDescriptors;
};
```

Now, lets initialize it when we initialize the swapchain and create these structs. Add this at the end of init_descriptors()

^code frame_desc chapter-4/vk_engine.cpp

And now, we can clear these every frame when we flush the frame deletion queue. This goes at the start of draw()

^code frame_clear chapter-4/vk_engine.cpp

Now that we can allocate descriptor sets dynamically, we will be allocating the buffer that holds scene data and create its descriptor set.

Add a new structure that we will use for the uniform buffer of scene data. We will hold view and projection matrix separated, and then premultiplied view-projection matrix. We also add some vec4s for a very basic lighting model that we will be building next.

```cpp
struct GPUSceneData {
    glm::mat4 view;
    glm::mat4 proj;
    glm::mat4 viewproj;
    glm::vec4 ambientColor;
    glm::vec4 sunlightDirection; // w for sun power
    glm::vec4 sunlightColor;
};
```

Add a new descriptor Layout on the VulkanEngine class
```
GPUSceneData sceneData;

VkDescriptorSetLayout _gpuSceneDataDescriptorLayout;
```

Create the descriptor set layout as part of init_descriptors. It will be a descriptor set with a single uniform buffer binding. We use uniform buffer here instead of SSBO because this is a small buffer. We arent using it through buffer device address because we have a single descriptor set for all objects so there isnt any overhead of managing it.

```cpp
{
	DescriptorLayoutBuilder builder;
	builder.add_binding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER);
	_gpuSceneDataDescriptorLayout = builder.build(_device, VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT);
}
```

Now, we will create this descriptor set every frame, inside the `draw_geometry()` function. We will also dynamically allocate the uniform buffer itself as a way to showcase how you could do temporal per-frame data that is dynamically created. It would be better to hold the buffers cached in our FrameData structure, but we will be doing it this way to show how. There are cases with dynamic draws and passes where you might want to do it this way.

```cpp	
	//allocate a new uniform buffer for the scene data
	AllocatedBuffer gpuSceneDataBuffer = create_buffer(sizeof(GPUSceneData), VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VMA_MEMORY_USAGE_CPU_TO_GPU);

	//add it to the deletion queue of this frame so it gets deleted once its been used
	get_current_frame()._deletionQueue.push_function([=, this]() {
		destroy_buffer(gpuSceneDataBuffer);
		});

	//write the buffer
	GPUSceneData* sceneUniformData = (GPUSceneData*)gpuSceneDataBuffer.allocation->GetMappedData();
	*sceneUniformData = sceneData;

	//create a descriptor set that binds that buffer and update it
	VkDescriptorSet globalDescriptor = get_current_frame()._frameDescriptors.allocate(_device, _gpuSceneDataDescriptorLayout);

	DescriptorWriter writer;
	writer.write_buffer(0, gpuSceneDataBuffer.buffer, sizeof(GPUSceneData), 0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER);
	writer.update_set(_device, globalDescriptor);
```

First we allocate the unifom buffer using the CPU_TO_GPU memory usage so that its a memory type that the cpu can write and gpu can read. This might be done on CPU RAM, but because its a small amount of data, the gpu is going to have no problem loading it into its caches. We can skip the logic with the staging buffer upload to dedicated gpu memory for cases like this.

Then we add it into the destruction queue of the current frame. This will destroy the buffer after the next frame is rendered, so it gives enough time for the GPU to be done accessing it. All of the resources we dynamically create for a single frame must go here for deletion.

To allocate the descriptor set we allocate it from the _frameDescriptors. That pool gets destroyed every frame, so same as with the deletion queue, it will be deleted automatically when the gpu is done with it 2 frames later. 

Then we write the new buffer into the descriptor set. Now we have the globalDescriptor ready to be used for drawing. We arent using the scene-data buffer right now, but it will be necessary later.

Before we continue with drawing, lets set up textures.

^nextlink

{% include comments.html term="Vkguide 2 Beta Comments" %}