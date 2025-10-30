---
layout: post
title: Razix Resource System 
tags: [C, Razix]
style: fill
color: warning
description: Razix engine new resource system rewritten in C99 compatible RHI API and how it interacts with engine high-level APIs 
blog: [Graphics]
published: true
permalink: /graphics/:title
---

Hi all, well...this will be one the first razix engine posts with deep dive into the architecture. I'll be talking about razix engine new C99 based RHI and resource system
that was re-written in C from C++. I was happy with C++ but a lot of bad decisions has led it into a unrecoverable tar and I wanted a clean slate to make some better decisions.

Re-writing it that too in C99 seemed like a better choice to make some good choices and correct some long standing architectural and implementations bugs
also if you can do it in all in C why not? No new heartbreaks so you'll be disappointed in that regards, no drama for the last few months so let's get straight into the post shall we.

## Razix Resources - new C99 API

Razix is a engine I've been making for the last 5 years, it's my portoflio and pratice sandbox, Recently I've decided to make a short game in it. I'll be getting into that later.
But since I'm looking to make something with it (as practice ofc not to sell or anything just to pratice as mike acton says but this time it's a lot of rendering systems for a game).

I've decided to correct some wrongs and make things neat and decoupled, especially the RHI API. Now the previous system was in C++, all gfx classed derived from `IRZResource` and high-level classes
and were buried deep in inheritance and was using virtual abstract classes to implement the backend and at one point I started hardcoding resource views in the classes.

Now the best way to fix this was re-write it and I thought why not make it C99 compatible. I also have a way with using macros with functions and code gen so I thought let's put that to use.

So I finally pulled the trigger and started re-writing it all in C99. I started off with DX12 this time as I was a vulkan driver dev, and I could write vulkan blindfolded,
I thought DX12 will give it a good nudge to balance the API, for ex by using RootSigs, Tables etc naming.

> Btw the engine is VK 1.3 minimum due to VK_KHR_dynamic_rendering, and descriptor_indexing extensions to support bindless rendering and I want things to be future proof than to be backward compatible.

Now main goal was to keep RHI as it's own DLL, logging/asserts everything independent from engine and a pure C99 library. So RHI has no state, it just exposes stuff to engine.
Maybe use some headers from Razix (rz_handle and profiling macros etc. but no source files at all). So the adventure begins and It took me around 2.5 months to get it all working.
All the Gfx tests (Triangle/texture/indirect draws etc.) passed and in the meantime I also added Linux support for the engine. 

I could finally add features like resource views, descriptor heaps backend memory allocations (ringbuffer vs freelist) to the new API. Also having all the backend 
code in a single file with a lot of RHI asserts helpmed me build a clean and safe API.

Now let's get into some code. Let's start off with rz_gfx_resource struct.

## rz_gfx_resource - Faking inheritance in C

let's see what it is and I'll try to defend my choices.

```C

RAZIX_RHI_ALIGN_16 typedef struct rz_gfx_resource
    {
        char                       pName[RAZIX_MAX_RESOURCE_NAME_CHAR];
        rz_handle                  handle;
        rz_gfx_resource_view_hints viewHints;
        rz_gfx_resource_type       type;
        rz_gfx_resource_state      currentState;
        uint8_t                    _pad0[4];

        union
        {
            rz_gfx_resource_view_desc    resourceViewDesc;
            rz_gfx_texture_desc          textureDesc;
            rz_gfx_sampler_desc          samplerDesc;
            rz_gfx_cmdpool_desc          cmdpoolDesc;
            rz_gfx_cmdbuf_desc           cmdbufDesc;
            rz_gfx_root_signature_desc   rootSignatureDesc;
            rz_gfx_descriptor_heap_desc  descriptorHeapDesc;
            rz_gfx_descriptor_table_desc descriptorTableDesc;
            rz_gfx_buffer_desc           bufferDesc;
            rz_gfx_shader_desc           shaderDesc;    // Big boi!
            rz_gfx_pipeline_desc         pipelineDesc;
        } desc;    // These are filled by the user, public members filled by backend are stored in each gfx_type separately
    } rz_gfx_resource;

```

Well uff, big chunkky boi right...

let's see how a resource is implemented in razix before I talk more:

```C

struct RAZIX_RHI_ALIGN_16 rz_gfx_cmdpool
    {
        RAZIX_GFX_RESOURCE;
        rz_gfx_cmdpool_type type;
        union
        {
#ifdef RAZIX_RENDER_API_VULKAN
            vk_cmdpool vk;
#endif
#ifdef RAZIX_RENDER_API_DIRECTX12
            dx12_cmdpool dx12;
#endif
        };
    };

    struct RAZIX_RHI_ALIGN_16 rz_gfx_cmdbuf
    {
        RAZIX_GFX_RESOURCE;
        union
        {
#ifdef RAZIX_RENDER_API_VULKAN
            vk_cmdbuf vk;
#endif
#ifdef RAZIX_RENDER_API_DIRECTX12
            dx12_cmdbuf dx12;
#endif
        };
    };

struct RAZIX_RHI_ALIGN_16 rz_gfx_shader
    {
        RAZIX_GFX_RESOURCE;
        rz_gfx_root_signature_handle rootSignature;
        uint32_t                     shaderStageMask;
        union
        {
#ifdef RAZIX_RENDER_API_VULKAN
            vk_shader vk;
#endif
#ifdef RAZIX_RENDER_API_DIRECTX12
            dx12_shader dx12;
#endif
        };
    };


```

Every struct start with a RZ_GFX_RESOURCE which is just
```C
#define RAZIX_GFX_RESOURCE rz_gfx_resource resource
```

So I wanted a way to fake inheritance, the easiest way is to embedd this `rz_gfx_resoruce` struct into every resource type of the RHI API.

> As for rz_handle please check Sebastial Altonnens tweets, it's a simple uint64_t handle to index into pool (More info here: 
> https://twitter.com/SebAaltonen/status/1534416275828514817 and https://twitter.com/SebAaltonen/status/1535175559067713536).

The resource structs has a name, a opaque handle to help index into resource pools and view hints to help with creations (backend APIs sometimes needs some flags to be set so we expose hints this way)
and state tracking is embedded at it's core. Now that about the bing union of desc structs?

## rz_gfx_resource - Faking public/private in C

I have 2 reasons to do this:
1. Fake public/private data in C (also helps with hot/cold data in future, read only rz_gfx_resource part of the data if needed)
2. My resource manager API in C++ uses macros to generate pools and a single signature to create/destroy and retieve resource so this solved that issue for me (I'll get into it more soon, just trust me for now)

Any generic data like say numSwapchainImages that the backend exposes can be stored inside rz_gfx_swapchain which is a resource where as any data that the user provides
to create the API handle is stored in the desc structs. So simple and clean boundary.

But why have them under a single union? Let me show how my resource manager and pools work and and you'll see the problem this solves.

## rz_gfx_resource - Resource Manager and macro magic...

Code speak louder so let's look at my resource manager API:

```C++

class RAZIX_API RZResourceManager : public RZSingleton<RZResourceManager>
        {
        public:
            /* Initializes the Resource System */
            void StartUp();
            /* Shuts down the Resource System */
            void ShutDown();

            /* Handles Resource Allocation functions */
            //-----------------------------------------------------------------------------------
            // Resource View
            RAZIX_REGISTER_RESOURCE_POOL(ResourceView, rz_gfx_resource_view)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(Texture, rz_gfx_texture)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(Sampler, rz_gfx_sampler)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(Shader, rz_gfx_shader)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(RootSignature, rz_gfx_root_signature)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(Pipeline, rz_gfx_pipeline)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(Buffer, rz_gfx_buffer)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(CommandPool, rz_gfx_cmdpool)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(CommandBuffer, rz_gfx_cmdbuf)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(DescriptorHeap, rz_gfx_descriptor_heap)
            //-----------------------------------------------------------------------------------
            RAZIX_REGISTER_RESOURCE_POOL(DescriptorTable, rz_gfx_descriptor_table)
            //-----------------------------------------------------------------------------------

        public:
            inline RZShaderBindMap& getShaderBindMap(const rz_gfx_shader_handle& shaderHandle)
            {
                return m_GlobalShaderBindMapRegistry[shaderHandle];
            }

        private:
            RZResourceCBFuncs                              m_ResourceTypeCBFuncs[RZ_GFX_RESOURCE_TYPE_COUNT];
            std::unordered_map<rz_handle, RZShaderBindMap> m_GlobalShaderBindMapRegistry;
        };


```

It's a small cute class, and uses some cool macros to generate the following API quickly:

```C++
// Resource Manager maintains the CPU pools for all resource allocated in Razix Engine

// API Usage:
// Resource Creation: get_RESOURCE_TYPE_NAME(args)
// Resource Destruction: destroy_RESOURCE_TYPE(handle)
// Resource Get RAW: getRESOURCE_TYPE_Resource(handle)
// ex. Draw Command Buffer
rz_gfx_cmdbuf_handle createDrawCommandBuffer(rz_gfx_cmdbuf_desc desc);
void                 destroyDrawCommandBuffer(rz_gfx_cmdbuf_handle handle);
rz_gfx_cmdbuf*       getDrawCommandBufferResource(rz_gfx_cmdbuf_handle handle);
```

So if you want to create a resource and get a simple uint64_t opeaue handle you pass it the desc struct and destory by passing the same handle, 
for getting the API pointer from the pool you pass the handle and get the resource. Again check seb's tweets for how handles work, I ported his C++ class
to a simple C API as so :

``` C++
// https://twitter.com/SebAaltonen/status/1534416275828514817
// https://twitter.com/SebAaltonen/status/1535175559067713536
/**
     * Handle is a weak pointer like reference to real objects inside a Pool, this forms the basis for various handles
     * 
     * Handles are like weak pointers. The container has an array of generation counters, and the data getter checks whether the generation counters match. 
     * If not, then the slot was deleted (and possibly reused). The getter returns null instead of the data pointer in this case
     */

#ifdef __cplusplus
extern "C"
{
#endif    // __cplusplus

    typedef struct rz_handle
    {
        uint32_t index;
        uint32_t generation;
    } rz_handle;

    static inline rz_handle rz_handle_create(uint32_t index, uint32_t generation)
    {
        rz_handle h = {index, generation};
        return h;
    }

    static inline rz_handle rz_handle_make_invalid()
    {
        rz_handle h = {0, 0};
        return h;
    }

    static inline uint32_t rz_handle_get_index(const rz_handle* handle)
    {
        return handle->index;
    }

    static inline void rz_handle_set_index(rz_handle* handle, uint32_t index)
    {
        handle->index = index;
    }

    static inline uint32_t rz_handle_get_generation(const rz_handle* handle)
    {
        return handle->generation;
    }

    static inline void rz_handle_set_generation(rz_handle* handle, uint32_t gen)
    {
        handle->generation = gen;
    }

    static inline bool rz_handle_is_valid(const rz_handle* handle)
    {
        return handle->generation != 0;
    }

    static inline void rz_handle_destroy(rz_handle* handle)
    {
        handle->index      = 0;
        handle->generation = 0;
    }

    static inline bool rz_handle_equals(const rz_handle* a, const rz_handle* b)
    {
        return a->index == b->index;
    }

    static inline bool rz_handle_not_equals(const rz_handle* a, const rz_handle* b)
    {
        return a->index != b->index;
    }

#ifdef __cplusplus
}
#endif    // __cplusplus

#ifdef __cplusplus
    #include <functional>

inline bool operator==(const rz_handle& a, const rz_handle& b) noexcept
{
    return a.index == b.index && a.generation == b.generation;
}

namespace std {
    template<>
    struct hash<rz_handle>
    {
        size_t operator()(const rz_handle& h) const noexcept
        {
            uint64_t x = (uint64_t(h.index) << 32) | uint64_t(h.generation);

            x += 0x9e3779b97f4a7c15ull;
            x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ull;
            x = (x ^ (x >> 27)) * 0x94d049bb133111ebull;
            x = x ^ (x >> 31);

            if constexpr (sizeof(size_t) == 8)
                return static_cast<size_t>(x);
            else
                return static_cast<size_t>(x ^ (x >> 32));
        }
    };
}    // namespace std
#endif
```

This is just for reference, now if you see all my resource pools use some macros to generate a consistent API for all gfx types:
1. All create functions take in a common desc structs
2. All destroy functions take a handle to free the resource from the resource pool (which btw is a simple freelist allocator)
3. All get resource takes in the handle to give the pointer handle from the free list pool

All `rz_gfx_resource` structs that derive use the same consistent API, wether you're a texture or buffer or command buffer. (with the exception of swapchain and syncobjs)

## The problem? - Caching public desc struct and consistent API
So what was the problem? how to pass the desc in a common way? 
- I could generate a set common functions and create API but they needed a desc struct, I also has to pass this struct to the backend API as it's the public data that the backend needs.
- and also store this in the rz_gfx_XX struct, which means all structs should have a desc member, so instead of doing it per struct I thought why not do it in `rz_gfx_resource` instead 
and use a union so that my RZResourceManager API can intercept the desc struct fill the union and pass it to backend and I don't have to maintain my rz_gfx_resource derived structs with desc,

I now have a central point to manage all my public data and also cache it (which was necessary) and also to maintain this consistent signature.

by using unions in the `rz_gfx_resource` I was finally able to solve this problem like so:
```C++

#define CREATE_UTIL(name, typeEnum, pool, handleSize)                                                    \
    rz_handle        handle;                                                                             \
    void*            where    = pool.obtain(handle);                                                     \
    rz_gfx_resource* resource = (rz_gfx_resource*) where;                                                \
    if (!where) {                                                                                        \
        RAZIX_CORE_ERROR("[Resource Manager] Failed to create resource: {0}. Pool is full!", name);      \
        return handle;                                                                                   \
    }                                                                                                    \
    memset(resource, 0x00, sizeof(rz_gfx_resource));                                                     \
    resource->type = typeEnum;                                                                           \
    RAZIX_CORE_TRACE("resourceName: {}", name);                                                          \
    snprintf(resource->pName, 256, "%s", name);                                                          \
    RAZIX_CORE_TRACE("resource->pName: {}", resource->pName);                                            \
    resource->handle = handle;                                                                           \
    memcpy(&resource->desc, &desc, handleSize);                                                          \
    if (m_ResourceTypeCBFuncs[typeEnum].createFuncCB) {                                                  \
        m_ResourceTypeCBFuncs[typeEnum].createFuncCB(where);                                             \
    } else {                                                                                             \
        RAZIX_CORE_ERROR("[Resource Manager] Resource Create Callback is NULL for resource: {0}", name); \
    }                                                                                                    \
    return handle;

```

As you can see I intercept the create functions and cache desc structs and pass it off to backend.

All the RHI API will use a single callback function with same signature to create/destroy resource as you will see next.

This is how the RZResourceManager implementation is generated:

```C++

void RZResourceManager::StartUp()
{
   RAZIX_CORE_INFO("[Resource Manager] Starting Up Resource Manager");
   //Razix::RZSplashScreen::Get().setLogString("Starting Resource Manager...");

   // ~ 10MiB for Resource View Pool (160 bytes * 65536)
   RAZIX_INIT_RESOURCE_POOL(ResourceView, RZ_GFX_RESOURCE_TYPE_RESOURCE_VIEW, 65536, sizeof(rz_gfx_resource_view), rzRHI_CreateResourceView, rzRHI_DestroyResourceView);

   // Initialize all the Pools
   // clang-format off
   // resource name | type | count | element size | create and destroy function pointers
   RAZIX_INIT_RESOURCE_POOL(Texture,         RZ_GFX_RESOURCE_TYPE_TEXTURE,          2048,        sizeof(rz_gfx_texture),            rzRHI_CreateTexture,         rzRHI_DestroyTexture);
   RAZIX_INIT_RESOURCE_POOL(Sampler,         RZ_GFX_RESOURCE_TYPE_SAMPLER,          64,          sizeof(rz_gfx_sampler),            rzRHI_CreateSampler,         rzRHI_DestroySampler);
   RAZIX_INIT_RESOURCE_POOL(Shader,          RZ_GFX_RESOURCE_TYPE_SHADER,           512,         sizeof(rz_gfx_shader),             RZSFCreateOverrideFunc,      DestroyShaderWithRootSigOverrideFunv);
   RAZIX_INIT_RESOURCE_POOL(RootSignature,   RZ_GFX_RESOURCE_TYPE_ROOT_SIGNATURE,   512,         sizeof(rz_gfx_root_signature),     rzRHI_CreateRootSignature,   rzRHI_DestroyRootSignature);
   RAZIX_INIT_RESOURCE_POOL(Pipeline,        RZ_GFX_RESOURCE_TYPE_PIPELINE,         512,         sizeof(rz_gfx_pipeline),           rzRHI_CreatePipeline,        rzRHI_DestroyPipeline);
   RAZIX_INIT_RESOURCE_POOL(Buffer,          RZ_GFX_RESOURCE_TYPE_BUFFER,           4096,        sizeof(rz_gfx_buffer),             rzRHI_CreateBuffer,          rzRHI_DestroyBuffer);
   RAZIX_INIT_RESOURCE_POOL(CommandPool,     RZ_GFX_RESOURCE_TYPE_CMD_POOL,         32,          sizeof(rz_gfx_cmdpool),            rzRHI_CreateCmdPool,         rzRHI_DestroyCmdPool);
   RAZIX_INIT_RESOURCE_POOL(CommandBuffer,   RZ_GFX_RESOURCE_TYPE_CMD_BUFFER,       32 * 32,     sizeof(rz_gfx_cmdbuf),             rzRHI_CreateCmdBuf,          rzRHI_DestroyCmdBuf);
   RAZIX_INIT_RESOURCE_POOL(DescriptorHeap,  RZ_GFX_RESOURCE_TYPE_DESCRIPTOR_HEAP,  4096,        sizeof(rz_gfx_descriptor_heap),    rzRHI_CreateDescriptorHeap,  rzRHI_DestroyDescriptorHeap);
   RAZIX_INIT_RESOURCE_POOL(DescriptorTable, RZ_GFX_RESOURCE_TYPE_DESCRIPTOR_TABLE, 4096 * 64,   sizeof(rz_gfx_descriptor_table),   rzRHI_CreateDescriptorTable, rzRHI_DestroyDescriptorTable);
   // clang-format on
}

void RZResourceManager::ShutDown()
{
   RAZIX_CORE_ERROR("[Resource Manager] Shutting Down Resource Manager");

   // Destroy all the Pools
   ////////////////////////////////
   RAZIX_UNREGISTER_RESOURCE_POOL(Texture);
   RAZIX_UNREGISTER_RESOURCE_POOL(Sampler);
   RAZIX_UNREGISTER_RESOURCE_POOL(Shader);
   RAZIX_UNREGISTER_RESOURCE_POOL(RootSignature);
   RAZIX_UNREGISTER_RESOURCE_POOL(Pipeline);
   RAZIX_UNREGISTER_RESOURCE_POOL(Buffer)
   RAZIX_UNREGISTER_RESOURCE_POOL(CommandPool);
   RAZIX_UNREGISTER_RESOURCE_POOL(CommandBuffer);
   RAZIX_UNREGISTER_RESOURCE_POOL(DescriptorHeap);
   RAZIX_UNREGISTER_RESOURCE_POOL(DescriptorTable);
   ////////////////////////////////

   RAZIX_UNREGISTER_RESOURCE_POOL(ResourceView);
}

//-----------------------------------------------------------------------------------

RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(ResourceView, RZ_GFX_RESOURCE_TYPE_RESOURCE_VIEW, rz_gfx_resource_view);

//-----------------------------------------------------------------------------------

RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(Texture, RZ_GFX_RESOURCE_TYPE_TEXTURE, rz_gfx_texture);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(Sampler, RZ_GFX_RESOURCE_TYPE_SAMPLER, rz_gfx_sampler);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(Shader, RZ_GFX_RESOURCE_TYPE_SHADER, rz_gfx_shader);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(RootSignature, RZ_GFX_RESOURCE_TYPE_ROOT_SIGNATURE, rz_gfx_root_signature);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(Pipeline, RZ_GFX_RESOURCE_TYPE_PIPELINE, rz_gfx_pipeline);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(Buffer, RZ_GFX_RESOURCE_TYPE_BUFFER, rz_gfx_buffer);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(CommandPool, RZ_GFX_RESOURCE_TYPE_CMD_POOL, rz_gfx_cmdpool);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(CommandBuffer, RZ_GFX_RESOURCE_TYPE_CMD_BUFFER, rz_gfx_cmdbuf);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(DescriptorHeap, RZ_GFX_RESOURCE_TYPE_DESCRIPTOR_HEAP, rz_gfx_descriptor_heap);
RAZIX_IMPLEMENT_RESOURCE_FUNCTIONS(DescriptorTable, RZ_GFX_RESOURCE_TYPE_DESCRIPTOR_TABLE, rz_gfx_descriptor_table);

//-----------------------------------------------------------------------------------

```

Again a consistent macro based API to quickly generate implementation, this reduces maintainence (debugging can get messy) but code looks super clean and 
users can use this super simple API with ease. No drama just a simple coherent pattern for the entire Gfx API.

## RHI API - How to create resources?

Because I have pass the rz_gfx_resource and memory to my backend API, all my RHI implementation get a chunk of memory to deal with like so:

```C
typedef void (*rzRHI_CreateSwapchainFn)(void* where, void*, uint32_t, uint32_t);
typedef void (*rzRHI_DestroySwapchainFn)(rz_gfx_swapchain*);

typedef void (*rzRHI_CreateCmdPoolFn)(void* where);
typedef void (*rzRHI_DestroyCmdPoolFn)(void* ptr);

typedef void (*rzRHI_CreateCmdBufFn)(void* where);
typedef void (*rzRHI_DestroyCmdBufFn)(void* ptr);

typedef void (*rzRHI_CreateShaderFn)(void* where);
typedef void (*rzRHI_DestroyShaderFn)(void* ptr);
```
So now I also have a clean API that takes in a simple rz_gfx_resource blob of memory to fill it. I think looking at the implementation will expplain itself:

I use jump tables to switch b/w different APIs and call the create/destroy CB functions, g_RHI jumptable does the honors.

```C
static void vk_CreateTexture(void* where)
{
    rz_gfx_texture* texture = (rz_gfx_texture*) where;
    RAZIX_RHI_ASSERT(rz_handle_is_valid(&texture->resource.handle), "Invalid texture handle, who is allocating this? ResourceManager should create a valid handle");
    rz_gfx_texture_desc* desc = &texture->resource.desc.textureDesc;
    RAZIX_RHI_ASSERT(desc != NULL, "Texture descriptor cannot be NULL");
    RAZIX_RHI_ASSERT(desc->width > 0 && desc->height > 0 && desc->depth > 0, "Texture dimensions must be greater than zero");

    // Maintain a second copy of hints...Ahhh...
    texture->resource.viewHints = desc->resourceHints;

    // Create VkImage
    VkImageCreateInfo imageInfo = {
        .sType     = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
        .imageType = vk_util_translate_texture_type_image_type(desc->textureType),
        .extent    = {
               .width  = desc->width,
               .height = desc->height,
               .depth  = desc->depth},
        .mipLevels     = desc->mipLevels,
        .arrayLayers   = (desc->textureType == RZ_GFX_TEXTURE_TYPE_CUBE) ? 6 : desc->arraySize,
        .format        = vk_util_translate_format(desc->format),
        .tiling        = VK_IMAGE_TILING_OPTIMAL,
        .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
        // FIXME: Optimize this based on usage flags for transfer src/dst, we already have these just use them here
        .usage       = VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT,
        .samples     = VK_SAMPLE_COUNT_1_BIT,
        .sharingMode = VK_SHARING_MODE_EXCLUSIVE};

    texture->resource.currentState = RZ_GFX_RESOURCE_STATE_UNDEFINED;
```

So this is how the new clean consitent API in razix works. An easy to learn API that works for C and C++

## More macro magic in C 

But I pass handles from C++ land and my C API needs raw pointers how to handle this: Let's look at this snippet.

I use macros to wrap over my g_RHI.XXX dispatch table to create functions for 2 reasons:

1. Create profiling/non-profilig macros for C/C++ land
2. Create handles and a pure C API versions;

I think the below code expains it better:

See how I wrap my RHI jumptable dispatches with macros:

```C++
#define rzRHI_DestroyDescriptorHeap  g_RHI.DestroyDescriptorHeap
#define rzRHI_CreateDescriptorTable  g_RHI.CreateDescriptorTable
#define rzRHI_DestroyDescriptorTable g_RHI.DestroyDescriptorTable


#if !RZ_PROFILER_ENABLED
    #if defined(RAZIX_RHI_USE_RESOURCE_MANAGER_HANDLES) && defined(__cplusplus)
        #define rzRHI_BeginCmdBuf(cb)              g_RHI.BeginCmdBuf(RZResourceManager::Get().getCommandBufferResource(cb))
        #define rzRHI_EndCmdBuf(cb)                g_RHI.EndCmdBuf(RZResourceManager::Get().getCommandBufferResource(cb))
    #else
        #define rzRHI_BeginCmdBuf                 g_RHI.BeginCmdBuf
        #define rzRHI_EndCmdBuf                   g_RHI.EndCmdBuf
 #else
    #if defined(RAZIX_RHI_USE_RESOURCE_MANAGER_HANDLES) && defined(__cplusplus)
         #define rzRHI_BeginCmdBuf(cb)                                                            \
             do {                                                                                 \
                 RAZIX_PROFILE_SCOPEC("rzRHI_BeginCmdBuf", RZ_PROFILE_COLOR_RHI_COMMAND_BUFFERS); \
                 g_RHI.BeginCmdBuf(RZResourceManager::Get().getCommandBufferResource(cb));        \
             } while (0)

         #define rzRHI_EndCmdBuf(cb)                                                            \
             do {                                                                               \
                 RAZIX_PROFILE_SCOPEC("rzRHI_EndCmdBuf", RZ_PROFILE_COLOR_RHI_COMMAND_BUFFERS); \
                 g_RHI.EndCmdBuf(RZResourceManager::Get().getCommandBufferResource(cb));        \
             } while (0)
  #endif
#else
      // C mode or direct handles mode with profiling support where available
        #define rzRHI_BeginCmdBuf(cb)                                                            \
            do {                                                                                 \
                RAZIX_PROFILE_SCOPEC("rzRHI_BeginCmdBuf", RZ_PROFILE_COLOR_RHI_COMMAND_BUFFERS); \
                g_RHI.BeginCmdBuf(cb);                                                           \
                RAZIX_PROFILE_SCOPEC_END();                                                      \
            } while (0)

        #define rzRHI_EndCmdBuf(cb)                                                            \
            do {                                                                               \
                RAZIX_PROFILE_SCOPEC("rzRHI_EndCmdBuf", RZ_PROFILE_COLOR_RHI_COMMAND_BUFFERS); \
                g_RHI.EndCmdBuf(cb);                                                           \
                RAZIX_PROFILE_SCOPEC_END();                                                    \
            } while (0)

  #endif    // defined(RAZIX_RHI_USE_RESOURCE_MANAGER_HANDLES) && defined(__cplusplus)
#endif    // !defined(RZ_PROFILER_ENABLED)

// Macro magic to the rescue!
```

Now this looks beautiful doesn't it, idk why I'm in soo much love with this way:
- It helps reduce jumptable injection for creating different version of the same API, I can just switch b/w 4 different version of API signatures
based on wether I'm in C/C++ land and wether I want to use Handles or not and to enable profiling etc. I offers free const functions.
Yes debuggin this is a pain in the ass but I have tight macros in backend to make sure things work as they should so I'm good with that regards. 

So this is the new C99 hybrid API for razix, pretty powerful and very flexible!!! checkout razix engine on my github for more implementation details.
I think it solved alot of porblems for and I hope I made a pretty good argument about the problems I face and how this API design solves the issues for
auto generating a clean and consistent API 

## Some problems - Reflection and markers
while SPIRV reflect is C, DX12 reflection and PIX markers are in C++ which is why I removed those features from RHI and added a few files like GfxUtil.h and GfxShaderUtils
that handle shader reflection and API markers and pass them to my RHI API. It's the engine responsibility to handle reflection and pass final info to my RHI, also use engine side macros to 
mark passes etc. So yeah that's how I deal with this to keep my RHI C99 only.

## ShaderBindMap for descriptor table management 
Now who manages the descriptor tables? I wanted a class to auto manage them:
- Handle automatic create/destruction by using reflection data from the shader.

So I came up with a class based on the Builder pattern, that uses reflection data to create tables and make sure user can bind them automatically.

THis is a simple class where you can add resources and views and create tables etc.

```C++

class RAZIX_API RZShaderBindMap
{
public:
   struct NamedResView
   {
       std::string                 name;
       rz_gfx_resource_view_handle resourceViewHandle;
   };

   enum BindMapValidationErr
   {
       BIND_MAP_VALIDATION_SUCCESS                 = 0,
       BIND_MAP_VALIDATION_DESCRIPTOR_MISMATCH     = 1 << 1,
       BIND_MAP_VALIDATION_BAD_TABLE_IDX           = 1 << 2,    // How do I use this?
       BIND_MAP_VALIDATION_INVALID_DESCRIPTOR      = 1 << 3,
       BIND_MAP_VALIDATION_FAILED                  = 1 << 4,
       BIND_MAP_VALIDATION_UNFULFILLED_DESCRIPTORS = 1 << 5,
   };

   RZShaderBindMap() = default;

   static RZShaderBindMap& RegisterBindMap(const rz_gfx_shader_handle& shaderHandle);
   static RZShaderBindMap& Create(void* where, const rz_gfx_shader_handle& shaderHandle);

   RZShaderBindMap& setResourceView(const std::string& shaderResName, const rz_gfx_resource_view& resourceView);
   RZShaderBindMap& setResourceView(const std::string& shaderResName, const rz_gfx_resource_view_handle& resourceViewHandle);
   RZShaderBindMap& setDescriptorTable(const rz_gfx_descriptor_table& descriptorTable);
   RZShaderBindMap& setDescriptorTable(const rz_gfx_descriptor_table_handle& descriptorTableHandle);
   RZShaderBindMap& setDescriptorBlacklist(const DescriptorBlacklist& blacklist);
   RZShaderBindMap& setDescriptorBlacklist(const std::string& name, const std::vector<std::string>& blacklistNames);
   RZShaderBindMap& validate();
   RZShaderBindMap& build();
   RZShaderBindMap& clear();
   RZShaderBindMap& clearBlacklist();
   RZShaderBindMap& destroy();

   void                 bind(rz_gfx_cmdbuf_handle cmdBufHandle, rz_gfx_pipeline_type pipelineType);
   BindMapValidationErr error();

   inline const rz_gfx_shader_reflection&                    getShaderReflection() const { return m_ShaderReflection; }
   inline const rz_gfx_shader_handle&                        getShaderHandle() const { return m_ShaderHandle; }
   inline const std::vector<rz_gfx_descriptor_table_handle>& getDescriptorTableHandles() const { return m_DescriptorTables; }
   inline const rz_gfx_descriptor_table_handle&              getDescriptorTableHandleAt(u32 idx) const { return m_DescriptorTables[idx]; }

private:
   rz_gfx_shader_reflection                           m_ShaderReflection        = {};
   rz_gfx_shader_handle                               m_ShaderHandle            = {};
   std::map<std::string, rz_gfx_resource_view_handle> m_ResourceViewHandleRefs  = {};
   std::vector<DescriptorBlacklist>                   m_BlacklistDescriptors    = {};
   std::map<u32, std::vector<NamedResView>>           m_TableBuilderResViewRefs = {};
   std::vector<rz_gfx_descriptor_table_handle>        m_DescriptorTables        = {};
   BindMapValidationErr                               m_LastError               = BIND_MAP_VALIDATION_FAILED;
   rz_gfx_root_signature_handle                       m_RootSigHandle           = {};
   union
   {
       u32 statusFlags = 0xffffffff;
       struct
       {
           u32 validated : 1;
           u32 built : 1;
           u32 dirty : 1;
       };
   };

private:
   RAZIX_NONCOPYABLE_IMMOVABLE_CLASS(RZShaderBindMap);

   RZShaderBindMap(rz_gfx_shader_handle shaderHandle);
};
```

The class goes thorught the reflection data, extract the root signature and check with descriptors and resource views using Names (Must match with those in shader)
to check if user has privided everything or not and build the final tables.

You can just call bind to bind the descriptor tables/sets and get going.

## Resource Views in FrameGraph

I also simplifies using resource view by embedding into per pass, everytime a resource is read/written in a framegraph pass we can provide it a `rz_gfx_resource_view_desc`.

It then lazily creates the resource and it's view and you use them during runtime to build tables once (during first frame etc.) and bind them.

(I plan to use a GPUTrain in future for GPU driven rendering and ditch the frame graph, but this should work with that too...)

So yeah everything is auto managed now yaaay!!!


## Closing notes
I hope this makes sense, It took me closer to 3 months and to get all tests pass properly and work on all platforms but this new API helped me fix a lot of things like
using a dedicated syncobj for each swapchain, asserts in backend, fix submission and presentation by not hiding state in RHI. RHI is now finally stateless like I wanted,
does 0 to none meomry allocs except for descriptor management in DX12 and importantly resource views! + and it's in C99. I'm happy with how it turned out,
this will be merged into master soon once I restore PBR rendering but overall a worthy refactor and great practice. Was it worth the 3 months? Definitely, and it gave me a great
base for consoles and metal. And writing is C gives no better dopamine hit. So yeah I could finally land the first draft before I start my new Job. Until next time ciao!! 
