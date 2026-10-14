+++
date = '2026-10-13T19:36:19-04:00'
draft = false
title = 'First Shapes'
showWordCount = false

tags = ["Rendering", "Core", "Platform"]
+++

{{< button pageRef="devlogs/little-engine/" target="_self" >}}
Return
{{< /button >}}

## Introduction

Last week, we left off with a simple code loop, which was the following.

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

In this article, we'll start from this basic loop to, first, do a bit of setup on the engine's architecture and, second, display our first shapes on screen.

Before changing anything, I think it's worth explaining a little of what's already present in the base loop, which I didn't explain in the previous article.

## Base Loop

The **main.cpp** file will represent, for now, the application that will use our engine. We'll call this application **Little Launcher**!

### Initializing the SDL Library

In the loop, we start by initializing a library. The **SDL** library (SDL3 to be exact).

```C++
// main.cpp

int main(int argc, char **argv) {
    
    SDL_Init(0);    
```

SDL notably helps us abstract functionality related to windows, inputs, and rendering. This saves us from having to handle the different OSes (e.g., Win32, Posix) or the different graphics APIs (e.g., DirectX, Vulkan, Metal).

I'll be relying on it for the early stages of development, and I'll gradually replace it with my own implementations later.

You may notice I don't deinitialize the library at the end of the loop — that's an oversight on my part. Currently, the program doesn't properly release its resources.

### Creating the Window

Next, I create a window by giving it a title and dimensions.

```C++
// main.cpp

    SDL_Window* window = SDL_CreateWindow("LittleLauncher", 1600, 900, 0); 
```

### Event Handling

Finally, we have the application's infinite loop.

We go through all the events that happened between the previous frame and this one. If one of them is a window-close event or a program-quit request, we stop the program.

```C++
// main.cpp

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
```

So, we have a small application that opens a black window. But the application isn't supposed to talk to SDL directly.

It should go through our engine, and therefore only talk to our engine's classes, which will act as a layer of abstraction over SDL.

## "Core" Module

The **Core** module will house our first classes, central to the engine.
It's its "core" (pun intended).

### Export

Each module will contain its export file. I'll show the Core module's as an example.

```C++
// Export.hpp

#pragma once

#ifdef LE_CORE_COMPILE
#define LE_CORE_API __declspec(dllexport)
#else
#define LE_CORE_API __declspec(dllimport)
#endif
```

This small file defines two macros that I'll place notably at the declaration level of my classes. They tell the compiler two things:
- When compiling the Core module (with LE_CORE_COMPILE present), it should mark symbols with **__declspec(dllexport)** as needing to be exported, so they can later be used by other applications.
- When compiling an application that uses the Core module (LE_CORE_COMPILE is absent), **__declspec(dllimport)** indicates that the symbols come from an external dll (Core's).

### "Platform" Class

Let's start by *wrapping* the initialization and shutdown of the SDL library.
I created a **Platform** class.

The header looks like this:

```C++
// Platform.hpp

namespace Le
{
    class LE_CORE_API Platform    
    {
    public:
        Platform();
        Platform(const Platform& other) = delete;
        Platform(Platform&& other) = delete;
        ~Platform();

        Platform& operator=(const Platform& other) = delete;
        Platform& operator=(Platform&& other) = delete;
    private:
    };
}
```

In the source file, I define the constructor and destructor:

```C++
// Platform.cpp

Le::Platform::Platform()
{
    if (!SDL_Init(0)) throw std::runtime_error(fmt::format("SLD_Init() failed: {}", SDL_GetError()));
}

Le::Platform::~Platform()
{
    SDL_Quit();
}
```

In the constructor, I initialize SDL. I shut it down in the destructor.
The advantage of this class is that now I no longer need to remember to call SDL_Quit.

```C++
//main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
```

I create a variable of type Platform, SDL gets initialized. Once the main function's scope ends, "platform"'s destructor is called and SDL gets shut down. Perfect!

### "Window" Class

The next part of the loop to *wrap* is window creation.

I'm adding a **Window** class to the Core module.

```C++
//Window.hpp

struct SDL_Window;

namespace Le
{

    class LE_CORE_API Window    
    {
    public:
        Window() = delete;
        Window(const std::string& title, int width, int height);
        Window(const Window& other) = delete;
        Window(Window&& other) = delete;
        ~Window();

        Window& operator=(const Window& other) = delete;
        Window& operator=(Window&& other) = delete;
    private:
        SDL_Window* m_handle;
    };
}
```

I added the constructor with a title and dimensions, and removed the default constructor.

```C++
// Window.cpp

Le::Window::Window(const std::string& title, int width, int height):
m_handle(nullptr)
{
    m_handle = SDL_CreateWindow(title.c_str(), width, height, 0);
}

Le::Window::~Window()
{
    if (m_handle)
    {
        SDL_DestroyWindow(m_handle);
    }
}
```

In the constructor, I create the SDL window and keep a pointer to it. I make sure to destroy it in the destructor to free resources. Same principle as with Platform.

Note that I had also forgotten, in the main loop, to destroy the window. I wrote the loop a bit too fast and wasn't paying attention.

Now let's replace it in main.

```C++
// main.cpp

    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
```

### Events

For events, the principle stays the same. Since things repeat, I'll only show the headers for certain struct definitions or enumerations.

First the **Event** class:

```C++
// Event.hpp

    struct  LE_CORE_API Event    
    {
    public:
        Event() = default;
        Event(const Event& other) = delete;
        Event(Event&& other) noexcept = default;
        ~Event() = default;

        Event& operator=(const Event& other) = delete;
        Event& operator=(Event&& other) noexcept = default;
        
        EventType eventType;
    };

    enum class LE_CORE_API EventType : std::uint32_t
    {
        First,
        Quit,
        Terminating,
        LowMemory,
        WillEnterBackground,
        DidEnterBackground,
        // etc...
        // And quite a few other values to match all of SDL's events.
    }
```

Then the **EventManager** class. The constructor and destructor stay default. We just have a PollEvent function:

```C++
// EventManager.cpp

bool Le::EventManager::PollEvent(Event& event) const
{
    SDL_Event sdlEvent;
    if (!SDL_PollEvent(&sdlEvent)) return false;


    switch (sdlEvent.type)
    {
        case SDL_EVENT_FIRST                             : event.eventType = EventType:: First                       ; break;
        case SDL_EVENT_QUIT                              : event.eventType = EventType:: Quit                        ; break;
        case SDL_EVENT_TERMINATING                       : event.eventType = EventType:: Terminating                 ; break;
        // ...
        // In the middle, there's a case for each SDL event
        //...
        case SDL_EVENT_USER                              : event.eventType = EventType:: User                        ; break;
        case SDL_EVENT_LAST                              : event.eventType = EventType:: Last                        ; break;
    default:
        throw std::runtime_error("EventManager : Unhandled event type");
    }

    return true;
}
```

Finally, we replace it in the main loop, and here's the end result:

```C++
// main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
    Le::EventManager eventManager{};
    
    bool isRunning = true;
    while (isRunning)
    {
        Le::Event event;
        while (eventManager.PollEvent(event))
        {
            switch (event.eventType)
            {
            case Le::EventType::Quit:
                isRunning = false;
                break;
            case Le::EventType::WindowCloseRequested:
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

That's great! Now we're still displaying a black window, but with our own classes, hehe!!

There's one thing I'm not entirely happy with. In the Window class's header, we have a pointer to an SDL_Window and we forward-declare that type.
The issue for me with that is, technically, the application talking to the Core module can still see SDL through it, when it should never catch a glimpse of SDL at all.
We'll sort that out later.

## Drawing Our First Shapes

To be able to display something on screen, we're going to use **SDL_Renderer**.

### Creating the Renderer

For now, we'll create it in **main.cpp** to test things out.

{{< image src="little-engine/devlog/first-shapes/creation-SDLRenderer.png">}}

We can see that we need a pointer to an SDL_Window to create the renderer, but our Window class doesn't have a function for that. Let's add a public *getter* function.

```C++
// Window.cpp

SDL_Window* Le::Window::GetHandle()
{
    return m_handle;
}
```

We now have the following code to create the renderer:

```C++
// main.cpp

    SDL_Renderer* renderer = SDL_CreateRenderer(window.GetHandle(), nullptr);
    if (!renderer)
    {
        fmt::printf("SDL_CreateRenderer() failed: {}", SDL_GetError());
        return 0;
    }
```
{{< alert "circle-info">}}
*I use **fmt** for logging to the console.*
*It's a handy library, especially for string formatting.*
{{< /alert >}}

However, if I try to run the program, the console displays the message for when **renderer** isn't valid.

{{< image src="little-engine/devlog/first-shapes/creation-renderer-error.png">}}

This error caught me off guard for a few seconds. Up until this point, I'd been on fairly familiar ground. I was reproducing what I'd done during my engine course. The only difference is that in my course project, I was linking SDL dynamically (SDL3.dll), whereas here I'm doing it statically (SDL3.lib).

And that's where the problem lies. Here's what I believe I understood.

### Static Linking to SDL

In my engine course, I had decided to set things up like this:

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->C[SDL3.dll];
B[Core.dll]-->C
{{< /mermaid >}}

Both binaries linked to the same SDL instance since I was using the dll version. So even though I initialized SDL in the Core module's binary, Little Launcher's binary was talking to that same, already-initialized instance. No problem there.

But this time, I wanted to link SDL statically, so I wouldn't have to carry the dll around, and so the library's code would be fully part of my own source code.

Except, for the renderer test, the Core module isn't the only one linking with SDL. Little Launcher does too, to create a renderer. But since SDL is linked statically, each binary ends up talking to a different instance of SDL.

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->C[SDL3.lib];
B[Core.dll]-->D[SDL3.lib]
{{< /mermaid >}}

Core initializes its instance of SDL. But Little Launcher's instance stays uninitialized.

And I had completely forgotten about this detail.

**All calls to SDL need to be made from the same module!!**

However, I don't just want to create a **Renderer** class in the Core module. What I had in mind instead was making rendering its own, separate module. The **Rendering** module.

And we'd end up with the same problem.

{{< mermaid >}}
graph TD;
A[Rendering.dll]-->C[SDL3.lib];
B[Core.dll]-->D[SDL3.lib]
{{< /mermaid >}}

## "Platform" Module

To fix this problem, I decided I'd create a module responsible for talking to SDL. All calls to SDL would be made from this one. The **Platform** module.

Currently, we have this relationship graph between the binaries:

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Core.dll];
{{< /mermaid >}}

If we add the Platform module, and move all calls to SDL inside it, we end up with this:

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Core.dll];
A[LittleLauncher.exe]-->C[Platform.dll];
B[Core.dll]-->C[Platform.dll];
{{< /mermaid >}}

Alright, let's go. For the Platform class, it's simple, I just move it into the Platform module. For the other classes, it's a bit different...

### WindowImpl

I'm going to create a **WindowImpl** class, which is an exact copy of the Window class. WindowImpl will live in the Platform module.

{{< alert "circle-info">}}
*I won't show the header and the cpp. It's an identical copy of Window with a different class name.*
{{< /alert >}}

Next, I modify the Window class so it no longer holds a handle to an SDL_Window directly, but instead to a WindowImpl.

```C++
// Window.hpp

#pragma once
#include <memory>
#include <string>
#include <LittleEngine/Core/Export.hpp>

namespace Le
{
    class WindowImpl;
    
    class LE_CORE_API Window    
    {
    public:
        Window() = delete;
        Window(const std::string& title, int width, int height);
        Window(const Window& other) = delete;
        Window(Window&& other) noexcept;
        ~Window();

        Window& operator=(const Window& other) = delete;
        Window& operator=(Window&& other) noexcept;

        const WindowImpl* GetImpl() const;
    private:
        
        std::unique_ptr<WindowImpl> m_impl;
    };
}
```

I'm showing the header again to point something out. Earlier, I mentioned I didn't like that the application could see SDL through Window's header.

Now, communication with SDL happens in WindowImpl, and I'm taking the opportunity to fully abstract away the implementation in Window. No SDL types are visible in Window's header anymore. WindowImpl's header is only included in the .cpp file.

Of course, there's nothing groundbreaking about this pattern. We forward-declare WindowImpl so as not to include its header in the .hpp, which can also potentially lighten compile times.

But I was happy that a side effect of this change was better abstraction of SDL from the perspective of the application using my engine.

Anyway, back to business.

Here's Window's .cpp now:

```C++
Le::Window::Window(const std::string& title, int width, int height):
m_impl(std::make_unique<WindowImpl>(title, width, height))
{
}

Le::Window::Window(Window&& other) noexcept:
m_impl(std::move(other.m_impl))
{
}

Le::Window::~Window()
{
    if (m_impl)
        m_impl.reset();
}

Le::Window& Le::Window::operator=(Window&& other) noexcept
{
    m_impl = std::move(other.m_impl);
    
    return *this;
}

const Le::WindowImpl* Le::Window::GetImpl() const
{
    return m_impl.get();
}
```

I took the opportunity to add the move constructor and move assignment operator, as well as a *getter* for **m_impl**. I figured it might come in handy.

### Moving the Event-Related Classes

I move the Event class into the Platform module.

And for EventManager, I do the same exercise as with Window by creating an EventManagerImpl class in the Platform module. Same principle.

### RendererImpl

Now, we still have the same loop, but with all calls to SDL made from the Platform module.

We're going to do the same thing with SDL_Renderer and create a RendererImpl class in the Platform module.

```C++
// RendererImpl.hpp

namespace Le
{
    class WindowImpl;

    class LE_PLATFORM_API RendererImpl    
    {
    public:
        RendererImpl() = delete;
        RendererImpl(const WindowImpl& windowImpl);
        RendererImpl(const RendererImpl& other) = delete;
        RendererImpl(RendererImpl&& other) = delete;
        ~RendererImpl();

        RendererImpl& operator=(const RendererImpl& other) = delete;
        RendererImpl& operator=(RendererImpl&& other) = delete;
        
    private:
        SDL_Renderer* m_renderer;
    };
}
```

```C++
// RendererImpl.cpp

Le::RendererImpl::RendererImpl(const WindowImpl& windowImpl):
m_renderer(nullptr)
{
    m_renderer = SDL_CreateRenderer(windowImpl.GetHandle(), nullptr);
    
    if (!m_renderer)
        fmt::printf("SDL_CreateRenderer() failed: {}", SDL_GetError());
    
}

Le::RendererImpl::~RendererImpl()
{
    if (m_renderer)
        SDL_DestroyRenderer(m_renderer);
}
```

For now, we'll call this directly in the main function.

```C++
// main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
    Le::EventManager eventManager{};
    
    Le::RendererImpl renderer(*window.GetImpl());
```

Now, if we run the application, it works!

## Back to Rendering Shapes

The issue is fixed, so we can finally try displaying our first shapes.

I add a few functions to RendererImpl to do some simple rendering.

```C++
// RendererImpl.cpp

void Le::RendererImpl::RenderLine(float x1, float y1, float x2, float y2)
{
    SDL_RenderLine(m_renderer, x1, y1, x2, y2);
}

void Le::RendererImpl::RenderFillRect(float x, float y, float width, float height)
{
    SDL_FRect rect {.x = x, .y = y, .w = width, .h = height};
    SDL_RenderFillRect(m_renderer, &rect);
}

void Le::RendererImpl::RenderCircle(float cx, float cy, float radius, int segments)
{
    SDL_FPoint center{.x = cx, .y = cy};
        
    std::vector<SDL_FPoint> points(segments + 1);
        
    float angle = std::numbers::pi_v<float> * 2.0f / segments;
        
    for (int i = 0; i <= segments; i++)
    {
        float pointAngle = angle * i;
        points[i].x = center.x + radius * std::cos(pointAngle);
        points[i].y = center.y + radius * std::sin(pointAngle);
    }
    
    SDL_RenderLines(m_renderer, points.data(), points.size());
}

void Le::RendererImpl::Clear()
{
    SDL_RenderClear(m_renderer);
}

void Le::RendererImpl::Present()
{
    SDL_RenderPresent(m_renderer);
}

void Le::RendererImpl::SetRenderDrawColor(const uint8_t& r, const uint8_t& g, const uint8_t& b, const uint8_t& a)
{
    SDL_SetRenderDrawColor(m_renderer, r, g, b, a);
}

```

And here's main:

```C++
// main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
    Le::EventManager eventManager{};
    
    Le::RendererImpl renderer(*window.GetImpl());

    bool isRunning = true;
    while (isRunning)
    {
        Le::Event event;
        while (eventManager.PollEvent(event))
        {
            switch (event.eventType)
            {
            case Le::EventType::Quit:
                isRunning = false;
                break;
            case Le::EventType::WindowCloseRequested:
                isRunning = false;
                break;
            default:
                break;
            }
        }
        
        // Rendering
        renderer.SetRenderDrawColor(33, 33, 33, 255);
        
        renderer.Clear();
        
        // Rect
        renderer.SetRenderDrawColor(233, 0, 0, 255);
        renderer.RenderFillRect(336, 386, 128, 128);
        
        // Line
        renderer.SetRenderDrawColor(0, 233, 0, 255);
        renderer.RenderLine(736, 450, 864, 450); 
        
        // Circle
        renderer.SetRenderDrawColor(0, 0, 233, 255);
        renderer.RenderCircle(1200, 450, 64, 64);
        
        renderer.Present();
    }
    
    return 0;
}
```

A little picture of the result!!

{{< image src="little-engine/devlog/first-shapes/shapes.png" >}}

We can now move on to this week's last step: creating the **Rendering** module.

## Rendering Module

I create a Rendering module and a Renderer class that goes in it.

Then I do the same exercise as with Window. Renderer becomes a *wrapper* around RendererImpl.

```C++
Le::Renderer::Renderer(const Window& window):
m_impl(std::make_unique<RendererImpl>(*window.GetImpl()))
{
}

Le::Renderer::~Renderer()
{
    if (m_impl)
        m_impl.reset();
}

void Le::Renderer::RenderLine(float x1, float y1, float x2, float y2)
{
    if (m_impl)
        m_impl->RenderLine(x1, y1, x2, y2);
}

void Le::Renderer::RenderFillRect(float x, float y, float width, float height)
{
    if (m_impl)
        m_impl->RenderFillRect(x, y, width, height);
}

void Le::Renderer::RenderCircle(float cx, float cy, float radius, int segments)
{
    if (m_impl)
        m_impl->RenderCircle(cx, cy, radius, segments);
}

void Le::Renderer::Clear()
{
    if (m_impl)
        m_impl->Clear();
}

void Le::Renderer::Present()
{
    if (m_impl)
        m_impl->Present();
}

void Le::Renderer::SetRenderDrawColor(const uint8_t& r, const uint8_t& g, const uint8_t& b, const uint8_t& a)
{
    if (m_impl)
        m_impl->SetRenderDrawColor(r, g, b, a);
}
```

In main, I replace the RendererImpl-typed variable with the Renderer type and make the necessary adjustments.

I also take the opportunity to use the **"friend class"** concept to make the **Window::GetImpl** function private. In theory, Renderer is the only one who needs this function. Moreover, the application isn't supposed to have access to WindowImpl, since that would indirectly let it see SDL's types, which I don't want.

With the Rendering module, we now have the following relationships:

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Rendering.dll];
B[Rendering.dll]-->C[Core.dll];
A[LittleLauncher.exe]-->C[Core.dll];
B[Rendering.dll]-->D[Platform.dll];
C[Core.dll]-->D[Platform.dll];
{{< /mermaid >}}

## Conclusion

That's it for this week!

If you have any recommendations or advice on how to improve my devlog or my engine, feel free to reach out to me:
* On **[LinkedIn](https://www.linkedin.com/in/antoine-hanna/)**
* By email: {{< email email="mailto:ahanna.pro@gmail.com" text="ahanna.pro@gmail.com" subject="">}}

You can find everything on the project's repo:
{{< github repo="Haiito92/LittleEngine" showThumbnail=true >}}

Wishing you a good day/evening, and see you next week!