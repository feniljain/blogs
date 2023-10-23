# Understanding LD_PRELOAD

This post assumes basic knowledge of shared and dynamic libraries.

Hunlo! Few days ago while surfing the web I discovered a blog post on an interesting env var: LD_PRELOAD.

LD_PRELOAD can help you override function calls. Just that, simple. Wait can't I just re-implement those same calls in my own code? Well, yes you can. But what if you are given a static binary from some third party installation and you want to change it's behaviour in specific ways?

LD_PRELOAD takes in a shared object in which you can define the exact functions you want to preload. Formal man pages say:
```
A list of additional, user-specified, ELF shared objects
to be loaded before all others.  This feature can be used
to selectively override functions in other shared objects.
```

There are other ways to preload too, in order of handling they are:

- LD_PRELOAD env var
- --preload cmd-line option when invoking the dynamic linker
- /etc/ld.so.preload file

They all achieve the same goal of overriding functions. Internet wise man said: "Fuck Around, Find Out", so let's just do that.

I am using a linux box on my mac, you can use any linux system/emulator, etc. It should work the same everywhere.
We will make a program which opens a given file, reads few starting bytes, closes it and quits. For our main program, it should look something like this:

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int main() {
    printf("Let's try to open a file\n");

    int makefile_handle = open("~/Projects/ld_preload/Makefile", O_RDONLY);

    char buf[100];
    printf("First few bytes:\n");
    read(makefile_handle, buf, 100);
    if(!errno) {
        printf("Error occurred: %d", errno);
        return 1;
    }
    printf("%s\n", buf);

    close(makefile_handle);

    return 0;
}
```

Let's compile this program by running:
```
gcc open.c -o open_file
```

You can use any compiler/compiler driver to make a simple executable. Running above program should open specified file and print first few bytes of it. Now let's override open() function call to log what files we are opening 👀

We will use this program to do just that:

```C
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>

typedef int (*orig_open_f_type)(const char* pathname, int flags);

int open(const char *pathname, int flags, ...) {
    printf("Used open() function on filename: %s\n", pathname);

    orig_open_f_type orig_open;
    orig_open = (orig_open_f_type)dlsym(RTLD_NEXT, "open");
    return orig_open(pathname, flags);
}
```

A lot of jargon, don't worry about it, for now, just follow me, we will understand the code after seeing something cool xD
Let's compile this program using:

```C
gcc -shared -fPIC override_open.c -o override_open
```

And finally let's run:
```bash
LD_PRELOAD=~/Projects/ld_preload/override_open.so ./open_file
```

You should see output similar to:
```
Let's try to open a file
Used open() function on filename: ~/Projects/ld_preload/Makefile
First few bytes:
h{
```

Let's understand the program now:

First we define _GNU_SOURCE, it is used to enable use of RTLD_NEXT in our example.

Then we define a function pointer, with args same as open() function we want to override. Next we define an open function with same args, this is the function that will be called when main application tries to use open().

In our own open function, we firstly print what filename is accessed, this is also what you see in the output. Next we make a function pointer variable name orig_open, that just means original_open.

Next, we use dlsym call to find next symbol with "open" name. This will be our original open function from libc. Quoting the man pages:
```
dlsym is used to obtain address of a symbol from a shared object or excutable.
```

Passing RTLD_NEXT instructs dlsym to find next symbol after current object. In our case, we are already inside "open", so we instruct it to find next "open". This should ideally be libc's open.

And finally at last we call orig_open() with pathname passed by user, so that it calls original libc open function and program does not break.

That does not seem exciting? You maybe thinking I knew what file it is going to open, I manually specified it, but think about open_file being some third part library you don't have any idea about and you want to know if it accesses you super secret passwords file, well you can use this to do just that.

You can also override open function to only make it open a single specific file all the time. Want to pause the world? Override gettimeofday() and return hardcoded time everytime. Want to unrandomize some application's seemingly random state? Override random() function to just return 1 always :P

All this sounds fun, but there are a few catches in this too.
- One only has restricted usage of LD_PRELOAD in safe environments, for e.g. linux does not allow paths with slashes in LD_PRELOAD.
- Next is let's say an application decides to some other variant of open() or any call for that matter, libc is usually a maze due to it holding backwards compatibility everywhere and has many similar functions. One would have to override them all to get the complete picture.
- Another being let's say you have an application making syscalls directly, those cannot be overrided with this method.

And that's a basic introduction, in next post we will do the same with a small Rust program instead :)

Credits: https://rafalcieslak.wordpress.com/2013/04/02/dynamic-linker-tricks-using-ld_preload-to-cheat-inject-features-and-investigate-programs/ this post helped me a lot to understand basics of LD_PRELOAD. My post uses similar examples from parent blog post. But we will also be building up tad bit on it in upcoming blog posts :)
