Automatic variable cleanup using `with` statements in C
=======================================================

It's interesting that more C programmers don't use the `with` statement for
automatic resource cleanup.

The `with` statement looks like a `for` loop, but instead of initialization,
condition, and step parts, `with` has declaration, startup, and cleanup.

For example, you could automatically close a file using:

```c
void write_samples(const char *path, int numSamples, const float *samples)
{
    with (FILE *fp; fp = fopen(path, "wb"); fclose(fp)) {
        fprintf(fp, "%d\n", numSamples);
        if (ferror(fp))
            break;
        for (int i = 0; i < numSamples; i++)
            fprintf(fp, "%f\n", samples[i]);
    }
}
```

Know that if the startup condition fails, the cleanup code isn't called.

So, if `fopen` failed to open `path` for writing, `fp` would be `NULL` and
`fclose(fp)` would not get executed.

If you want to actually handle this situation, you could use the optional
`else` clause, which is described next.

```c
int show_window(void)
{
    with (; SDL_Init(SDL_INIT_VIDEO) >= 0; SDL_Quit())
    {
        with (SDL_Window *w; w = SDL_CreateWindow("With!", 0, 0, 320, 240, NULL); SDL_DestroyWindow(w))
        {
            SDL_Delay(3000);
        }
        else
        {
            SDL_Log("Failed to create window: %s", SDL_GetError());
            return 4;
        }
    }
    else
    {
        SDL_Log("Failed to reticulate splines: %s", SDL_GetError());
        return 3;
    }
    return 0;
}
```

Observe that `with`s can nest.

Also, any variables declared are still within scope in the `else` clause, but
usually they'll be `NULL`, so they have little practical use.

Amazingly, you can `return` from them and any cleanup code will still run.

In conclusion, like all constructs in C, `with` is not always the right
tool to use, but for simple resource cleanup, it should be considered.

## Truth

OK, so if you're wondering why you've been programming in C since 1972 and
have _NEVER_ once heard of the `with` statement, you're not crazy.

**The `with`-statement doesn't exist**.

Typically, the type of resource cleanup I just described is done in C with
pairs of startup and cleanup macros, additional helper functions,
[compiler extensions](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization#Clang_and_GCC_%22cleanup%22_extension_for_C),
or `goto`s (if you don't mind arguing with every new developer that joins
your project about their appropriate uses).

The `with`-statement seems appropriately C though.
Unlike destructors in object-oriented languages, the cleanup code is explicit
and easier to understand what is actually happening as it doesn't hide the
control flow.

The optional `else`-clause used when startup fails, looks as appropriate at the
end of the `with` as it does after an `if`, so it wouldn't require its own
unique keyword.

Finally, some compilers already look for extensions like the `cleanup`
attribute, so `with` doesn't seem that much more complex to implement.

I hope someone will read this and we'll see the addition of `with` in the next
C standard, but until then feel free to use [`with.h`](with.h).

## Using `with.h`

[`with.h`](with.h) defines two macros `with` and `withif` that look pretty
close to the syntax I introduced.
They call code before and after another block of code.

You need only `#include "with.h"` in your code.

### Macro `with(declare, startup, cleanup, block)`

```c
#include "with.h"

// Head prints the first 128 characters of the file at `path`.
void Head(const char *path)
{
    char buf[128] = {0};
    int success = 0;

    with (FILE *fp, fp = fopen(path, "rb"), fclose(fp), {
        fread(buf, 1, sizeof(buf)-1, fp);
        if (ferror(fp)) {
            perror(path)
            break;
        }
        printf("%s\n", buf);
        success = 1;
    })

    if (success)
        return;

    fputs("Well... we tried.\n", stderr);
}
```

### Macro `withif(declare, startup, cleanup, block, otherwise)`

Use `withif` if you need the `else`.
Here's the `show_window` example from before:

```c
int show_window(void)
{
    int result = 0;
    withif (, SDL_Init(SDL_INIT_VIDEO) >= 0, SDL_Quit(), {
        withif (SDL_Window *w, w = SDL_CreateWindow("With!", 0, 0, 320, 240, NULL), SDL_DestroyWindow(w),
        {
            SDL_Delay(3000);
        },
        else
        {
            SDL_Log("Failed to create window: %s", SDL_GetError());
            result = 4;
        })
    },
    else
    {
        SDL_Log("Failed to reticulate splines: %s", SDL_GetError());
        result = 3;
    })
    return result;
}
```

### Caveat

* Do not use `return`. Sorry, but the macros are not as powerful as the builtin
`with`-statement would be.

* Use nested `with`s for multiple variables.
    ```
    with (FILE *reader, reader = fopen("input", "r"), fclose(reader), {
        with (FILE *writer, writer = fopen("output", "w"), fclose(writer), {
            ...
        })
    })
    ```

* If you *have* to declare multiple variables, you'll need to wrap them in
  parenthesis.
  Since these are macro functions, we have to use commas to separate the
  parameters.
  ```
  with (
      (FILE *reader, *writer),
      (reader = fopen("input", "r"); writer = fopen("output", "w")),
      (fclose(reader); fclose(write)),
      { ... }
  )
  ```

