+++
date = '2026-10-02T17:35:54-04:00'
draft = false
title = 'Hello World'
showWordCount = false
+++

## Introduction

**"Hello World!"**. This is the very first post of my devlog for Little Engine.

Little Engine is, for me, a project that's above all about having fun, because I really enjoy getting into low-level code.
At the same time, it's also a learning and discovery project.

The idea is to first build a small engine with the help of common libraries.
SDL, Box2D, Jolt, etc...

Then, one by one, replace the middlewares/libraries with my own integrations built from scratch!

I plan to post at a pace of one post per week. Sometimes the progress will be big, sometimes small. Everything will be shared with you!

## First step: Hello world

This week, I simply wrote a small piece of code, using **SDL3**, to open a window that can be closed.

```C++
// main.cpp

#include <SDL3/SDL.h>

int main(int argc, char **argv) {
    
    SDL_Init(0);    
    
    SDL_Window* window = SDL_CreateWindow("LittleLauncher", 1600, 900, 0);
    
    bool isRunning = true;
    while (isRunning)
    {
        SDL_Event event;
        while (SDL_PollEvent(&event))
        {
            switch (event.type)
            {
            case SDL_EVENT_QUIT:
                isRunning = false;
                break;
            case SDL_EVENT_WINDOW_CLOSE_REQUESTED:
                isRunning = false;
                break;
            default:
                break;
            }
        }
    }
    
    return 0;
}
```

## Xmake Configuration

This week's engine code boils down to the small script above.
The rest of my work this week was mostly a bit of setup for the **Xmake** config.

Xmake is a **build tool**. It allows you to create configuration files that make it easier to handle:
* Project generation
* Package management
* Compilation management
* Building/packaging


I set up the xmake.lua (the main config file), as well as two "task" files to make creating modules and classes easier.

### Config file: xmake.lua

The config file handles compilation, module separation, targets, etc...

```lua
-- xmake.lua

----- tasks -----
includes("xmake/createclass.lua")
includes("xmake/createmodule.lua")

----- modules -----

add_rules("mode.debug", "mode.release")
set_languages("c++23")

add_requires("libsdl3")

add_includedirs("inc")

modules = {
    Core = {
        Defines = {"LE_CORE_COMPILE"},
        Packages = {"libsdl3"}
        }
    }

for name, module in pairs(modules) do
    target("LittleEngine" .. name)
        set_group("LittleEngine")
        set_kind("shared")
        
        if module.Defines then
            add_defines(table.unpack(module.Defines))
        end
    
        if module.Packages then
            add_packages(table.unpack(module.Packages))
        end
        
        add_headerfiles("inc/(LittleEngine/" .. name .. "/**.hpp)")
        add_headerfiles("inc/(LittleEngine/" .. name .. "/**.inl)")
        add_files("src/LittleEngine/" .. name.. "/**.cpp")
    end

target("LittleLauncher")
    add_packages("libsdl3")
    set_kind("binary")
    add_files("src/main.cpp")

```

### Xmake Tasks

The "task" files let me create my own custom xmake commands to carry out various tasks.

I can create a .lua file (which I then import into the main config file), and declare a task. Here's the class-creation task as an example:

```lua
-- createclass.lua

task("create-class")
    
    set_menu {
        usage = "xmake create-class name [options]",
        description = "Creates files for a new class.",
        options = {
            {'n', "name", "kv", nil, "The name of the class to create."},
            {'m', "module", "kv", nil, "The module of the class."},
            {nil, "noinl", "k", nil, "For classes with no inl file."},
            {nil, "nocpp", "k", nil, "For classes with no cpp file."}
        }
    }

    on_run(function()

    -- Body not shown, it's just string templating.

    end)
```

Through set_menu, I tell xmake how to call the task.
I can then run it from the command prompt.

```pwsh
xmake create-class -n Application -m Core
```

## Conclusion

That's it for this week!

If you have any recommendations or advice on how to improve my devlog or my engine, feel free to reach out to me:
* On **[LinkedIn](https://www.linkedin.com/in/antoine-hanna/)**
* By email: {{< email email="mailto:ahanna.pro@gmail.com" text="ahanna.pro@gmail.com" subject="">}}

You can find everything on the project's repo:
{{< github repo="Haiito92/LittleEngine" showThumbnail=true >}}

Wishing you a good day/evening, and see you next week!