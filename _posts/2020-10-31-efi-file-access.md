---
layout: post
title: UEFI File access APIs
excerpt_separator: <!--more-->
tags: efi clang microsoft
---

[UEFI Specification]: https://uefi.org/sites/default/files/resources/UEFI%20Spec%202.8B%20May%202020.pdf "UEFI Specification"
[previous post]: {% post_url 2020-10-18-handles-guids-and-protocols %} "the previous post"


Continuing exploring UEFI bit by bit. This time I'm going to briefly cover the
file access APIs in UEFI. As usual all the sources are available on
[GitHub](https://github.com/krinkinmu/efi), however since the [previous post]
I've made a few changes in the repository that this post will not cover.

<!--more-->

# Simple File System Protocol

We will start with the Simple File System Protocol. [UEFI Specification] covers
this protocol in the section 13.4 Simple File System Protocol.

I find the name of the protocol somewhat misleading. The simple part is correct
because the interface of the protocol contains just one function:

```c
#define EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID \
    { 0x0964e5b22, 0x6459, 0x11d2, \
      { 0x8e, 0x39, 0x00, 0xa0, 0xc9, 0x69, 0x72, 0x3b } }

struct efi_simple_file_system_protocol {
    uint64_t revision;
    efi_status_t (*open_volume)(
        struct efi_simple_file_system_protocol *, struct efi_file_protocol **);
};
```

On the other hand, this function is not what I'd expect a File System intreface
to look like. Simple File System Protocol just provides access to the real file
system interface which is represented by the File Protocol.

One way to look at it is that Simple File System Protocol provides access to the
root directory of the file system on a device or partition of the device. That
begs a question how do we find a device though?

Here is the way I used in my example. In the [previous post] we discussed the
Loaded Image Protocol. This protocol among other things contains a handle of the
device the image was loaded from. This device might also support the file API,
so we can try to open the Simple File System Protocol on the device:

```c
// This function gets access to the Loaded Image Protocol given the application
// handle (the same handle that is passed to the efi_main).
efi_status_t get_image(
    efi_handle_t app,
    struct efi_system_table *system,
    struct efi_loaded_image_protocol **image)
{
    struct efi_guid guid = EFI_LOADED_IMAGE_PROTOCOL_GUID;

    return system->boot->open_protocol(
        app,
        &guid,
        (void **)image,
        app,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
}

// This function gets access to the Simple File System Protocol given the
// application handle (the same handle that is passed to the efi_main) and
// a handle of the device that supports the Simple File System Protocol.
efi_status_t get_rootfs(
    efi_handle_t app,
    struct efi_system_table *system,
    efi_handle_t device,
    struct efi_simple_file_system_protocol **rootfs)
{
    struct efi_guid guid = EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID;

    return system->boot->open_protocol(
        device,
        &guid,
        (void **)rootfs,
        app,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
}
```

# File Protocol

Now we have the Simple File System Protocol available to us we can use it to
get our hands of File Protocol. File Protocol is the actual file system API we
need to open/creat, read and write files. The File Protocol is described in
13.5 File Protocol of the [UEFI Specification].

Here is a somewhat shortened version of the File Protocol definition I used:

```c
// Open modes
static const uint64_t EFI_FILE_MODE_READ = 0x0000000000000001;
static const uint64_t EFI_FILE_MODE_WRITE = 0x0000000000000002;
static const uint64_t EFI_FILE_MODE_CREATE = 0x8000000000000000;

// File attributes
static const uint64_t EFI_FILE_READ_ONLY = 0x1;
static const uint64_t EFI_FILE_HIDDEN = 0x2;
static const uint64_t EFI_FILE_SYSTEM = 0x4;
static const uint64_t EFI_FILE_RESERVED = 0x8;
static const uint64_t EFI_FILE_DIRECTORY = 0x10;
static const uint64_t EFI_FILE_ARCHIVE = 0x20;

struct efi_file_info;

struct efi_file_protocol {
    uint64_t revision;
    efi_status_t (*open)(
        struct efi_file_protocol *,
        struct efi_file_protocol **,
        uint16_t *,
        uint64_t,
        uint64_t);
    efi_status_t (*close)(struct efi_file_protocol *);

    void (*unused1)();

    efi_status_t (*read)(struct efi_file_protocol *, efi_uint_t *, void *);

    void (*unused2)();
    void (*unused3)();
    void (*unused4)();

    efi_status_t (*get_info)(
        struct efi_file_protocol *, struct efi_guid *, efi_uint_t *, void *);

    void (*unused6)();
    void (*unused7)();
    void (*unused8)();
    void (*unused9)();
    void (*unused10)();
    void (*unused11)();
};
```

As was briefly mentioned in the previous section Simple File System Protocol
gives use File Protocol instance that corresponds to the root directory of the
file system. From there we could use `open` function to open other directories
and files.

> *NOTE:* as you could imagine a lot of [UEFI Specification] uses things coming
  from the Windows world. That is also the case when it comes to file paths.
  Specifically, file path separator in UEFI is `\`, which when you have to use
  it in C string or character literal have to be escaped, so it becomes `\\`.

Here is how you can obtain File Protocol instance (root of the file system)
from the Simple File System Protocol instance:

```c
efi_status_t get_rootdir(
    struct efi_simple_file_system_protocol *rootfs,
    struct efi_file_protocol **rootdir)
{
    return rootfs->open_volume(rootfs, rootdir);
}
```

Rather simple isn't it? Now let's take a look at a simple motivating example
that will show how to use File Protocol.

# File Information

One thing that might come useful when if you want to create an OS loader in
UEFI is to be able to load file in memory. In order to do that you should
know how much memory you need to load the kernel image.

Depending on the format of the image it might take some work to determine how
much memory you actually need for the kernel image. For now we will go a
somewhat simplified route and just check the kernel image size and assume that
it's what we need.

> *NOTE:* That's just for the purpose of the example in this post, I will, of
  course, try to correct that in the future.

To get basic file information we first have to open the file to get the
instance of File Protocol corresponding to the file we need. In this example
I will work with `efi\boot\bootx64.efi` - the file of our EFI application
image. Mostly because it already exists and I don't need to create anything
new.

> *NOTE:* `efi\boot\bootx64.efi` is the path starting from the root of the file
  system.

First we need to open that file. Let's assume that variable `rootdir` of type
`struct efi_file_protocol *` is our root directory, the pointer that we
obtained using `get_rootdir` function above. To open the file all we need to
do is:

```c
    struct efi_file_protocol *rootdir;
    struct efi_file_protocol *kernel;
    uint16_t path[] = u"efi\\boot\\bootx64.efi";
    efi_status_t status;

    // rootdir initialization need to happen here

    // We assume the file already exist and we open it in read only mode
    status = rootdir->open(
        rootdir, &kernel, path, EFI_FILE_MODE_READ, EFI_FILE_READ_ONLY);

    if (status != 0)
        // handle errors
        goto out;

    // here you can work with the file using the kernel pointer

out:
    // cleanup
```

Now we have an instance of File Protocol that represents the file we are
interested in. To get basic file information you can use the `get_info`
function. The interface of the function is somewhat terrible. It takes four
parameters:

1. the File Protocol pointer - nothing unexpected here;
2. a pointer to a GUID - it describes what kind of information you want from
   the function;
3. pointer to the size (I'll cover it further down);
4. a `void` pointer to the buffer where there results will be returned.

You see, `get_info` function can be used to request two kinds of information:

1. the actual file information (size, last access time, last modification
   time, etc);
2. general file system information (size, free space, label etc).

Why they couldn't use two different functions for that is not clear to me, but
it's how it is. Naturally, we have to tell the function what kind of
information we actually need. That's what the second parameter of the function
is for - we specify different GUIDs for different kinds of information we want.

In this example we only care anout the file information and not the general
file system information. Here are the definitions of GUID and the structure
used to return results for our case:

```c
#define EFI_FILE_INFO_GUID \
    { 0x09576e92, 0x6d3f, 0x11d2, \
      { 0x8e, 0x39, 0x00, 0xa0, 0xc9, 0x69, 0x72, 0x3b } }

struct efi_file_info {
    uint64_t size;
    uint64_t file_size;
    uint64_t physical_size;
    struct efi_time create_time;
    struct efi_time last_access_time;
    struct efi_time modifiction_time;
    uint64_t attribute;
    // The efi_file_info structure is supposed to be a variable size structure,
    // but it's really a pain to always dynamically allocate enough space for
    // the structure, so I explicitly allocated some space in the structure, so
    // we will be able to cover at least some simple cases without dynamic
    // memory allocation.
    uint16_t file_name[256];
};
```

Now the time has come for the less ugly part of the `get_info` interface, but
yet somehow surprisingly inconvenient. In the [UEFI Specification] the
`struct efi_file_info` is actually given as a variable length structure and the
size of the `file_name` array is not actually fixed like in the snippet above.

It kind of makes sense, because in general the file name might be arbitrary
long. Different file systems can put different limitations on the length of the
file name, but this part of the [UEFI Specification] is file system agnostic,
so it cannot set a specific limit.

Because of this, when calling `get_info` we have to specify the size of the
buffer we pass as a forth argument of `get_info`. That's what the thrid
argument is for. It also serves as a output argument and if the function is
successful it will contain the actual size of the structure.

> *NOTE:* note that the `struct efi_file_info` contains field `size`. This
  field has nothing to do with the file size, it's actually the size of the
  structure itself including the variable length part that contains the file
  name. So there was no real reason to turn the third argument of `get_info`
  into an output argument.

Because it's somewhat inconvenient to always dynamically allocate memory for
the `struct efi_file_info` I defined the structure slightly differently from
the way it's defined in the [UEFI Specification] and added allocated some
buffer for the file name right in the structure. I still can allocate memory
for the structure dynamically and exceed the 256 characters buffer allocated
in the structure, but in those cases when I know that the name of the file
fits into 256 characters I don't need to do that.

Well, all that's left is to show a simple example:

```c
    struct efi_file_protocol *rootdir;
    struct efi_file_protocol *kernel;
    struct efi_file_info file_info;
    efi_uint_t size;
    efi_status_t status;

    // rootdir and kernel initialization need to happen here,
    // see the example above

    size = sizeof(file_info);
    status = kernel->get_info(kernel, &guid, &size, (void *)&file_info);
    if (status != 0)
        // handle errors
        goto close;

    u16snprintf(
        buffer, sizeof(buffer),
        "file %w size %llu (%llu on disk) bytes\r\n",
        file_info.file_name,
        (unsigned long long)file_info.file_size,
        (unsigned long long)file_info.physical_size);

    system->out->output_string(system->out, buffer);

close:
    status = rootdir->close(kernel);
    
    // cleanup
```

> *NOTE:* `u16snprintf` is the function I created to do string formatting in
  a convenient way. I will not cover it in this post, but the implementation
  is available in the repository.

> *NOTE:* Coincendentally I noticed that it didn't matter whether I used
  `kernel` or `rootdir` to get `get_info` function pointer. Both happened to
  work the same way in my case, though I don't know if it's guaranteed.

# Instead of conclusion

UEFI File access APIs are not hard to use, even though some aspects of the API
design smell somewhat.
