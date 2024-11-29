---
title: What's up with `typename` and `const` positioning in C++
tags: [C++]
style: fill
color: danger
description: the confusion with typename and const placement order with different compilers 
blog: [Graphics]
published: true
permalink: /graphics/:title
---

Hi all, phani here, ready for another blog post? I was dealing with this fuckery🚧 when writing the frame graph for my engine and I had to deal with Nested types and templates and 
I ran into some fun issues wiile using `typename` and `const` together with MSVC and Clang/GCC. In my engine I store the struct to create a resource as it's nested type and when 
the frame graph builder takes this type and it's `T::CreateDesc` as r-value refernces (&&) and stores it. 
That's the basic idea and I store the nested CreateDesc struct as a const member along with the resource `T`.

We have a nested type called CreateDesc inside TextureResource, now this struct will be common name for all types of resources.
Which means we can have a common `T::Desc getResourceDescription()` function without needing virtual functions or common types we can resolve the members during compile time.
This is very powerful and extremely **type-safe** without the need for any complex setup.

Seems simple and acceptable I guess. But the order of template and const in MSVC is different from Clang/GCC, as usual MSVC being weak didn't warn and I spent many hours debugging 100s of errors
I got all over my engine while porting. So let's dive deep to see what's happening here.

Btw, the whole points of using templates here is to store the different types of resources using TypeErasure and not to deal with Interfaces 
and all, you can read more about type erasure here: 
- https://davekilian.com/cpp-type-erasure.html
- https://www.youtube.com/watch?v=iMzEUdacznQ

## The Problem

consider this snippet of code
```cpp

struct TextureResource
{
    struct CreateDesc
    {
        int Width = 0;
        int Height = 0;
    };
    int HandleIdx = 0;
};

template <typename T>
class FrameGraphTypeErasedResource
{
public:
    FrameGraphTypeErasedResource(typename T::CreateDesc&& val, T&& obj)
        : descriptor(std::move(val)), resource(std::move(obj)) {}

    typename const  T::CreateDesc descriptor;
    T resource;
};

int main()
{
    TextureResource dummyTexture;
    dummyTexture.HandleIdx= 1;

    TextureResource::CreateDesc textureCreateDesc{};
    textureCreateDesc.Width = 100;
    textureCreateDesc.Height = 100;
    
    FrameGraphTypeErasedResource<TextureResource> ins((TextureResource::CreateDesc)textureCreateDesc, (TextureResource)(dummyTexture));
    
   return 0;
}

```

> Use these for quick testing/benchmarking on different compilers when dealing with issues and you don't have to frequently switch over different machines.
> - https://godbolt.org
> - https://cppinsights.io
> - https://quick-bench.com

## MSVC vs Clang
This code compile fine on MSVC, you can check here on godbolt: https://godbolt.org/z/dEbEcsYe3, but it it will fail on clang/gcc compilers with the following errors

> expected a qualified name after 'typename' typename const T::CreateDesc descriptor;

Fun error right? actually very disturbing...the problem comes down to the order of the keywords `typename` and `const`.

**Clang treats the typename as unnecessary or misplaced because `const` qualifier is used to modify types, not dependent names**.

Since `const` is applied on types and and T::CreateDesc has not been resolved as a type yet until we prefix it with the keyword `typename` first, so it won't work and clang/gcc rightfuly throws proper erros unlike MSVC.

Clang expects `const typename T::CreateDesc` instead, following the standard rule where const comes first when applied to a type derived from a template parameter.

## Closing Notes

### Resolve the type first before applying any qualifiers on them.

If I made any mistakes or wrong assumptions or you would like to correct anything reach out to me at phani.s2909@icloud.com or on Github: https://github.com/Pikachuxxxx. Thanks.

## Life Update
So it's been just a few days since my last post and yeah tbh nothing much has been happening in my life yet, just exploring random code and watching some TV Shows and dusting my PS5.
So I started watching Sucession again, first time it I binged it but this time, knowing the ending and with nothing on my mind, makes it very interesting and enjoyable...amazing show and writing.
And as usual my fav character is the no.1️⃣ boy - _L to the "OG"_ Kendall Roy 🧢😃.

