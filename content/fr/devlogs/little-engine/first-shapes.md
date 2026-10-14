+++
date = '2026-10-13T19:36:19-04:00'
draft = false
title = 'Premières formes'
showWordCount = false

tags = ["Rendering", "Core", "Platform"]
+++

{{< button pageRef="devlogs/little-engine/" target="_self" >}}
Retour
{{< /button >}}

## Introduction

La semaine dernière, nous nous étions laissés sur une boucle de code simple qui était la suivante.

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

Dans cet article, on va partir de cette boucle de base pour, dans un premier temps, faire un peu de mise en place de l'architecture du moteur et, dans un second temps, afficher nos premières formes à l'écran.

Avant de modifier quoi que ce soit, je pense qu'il convient d'expliquer un petit peu ce qui est déjà présent dans la boucle de base, que je n'ai pas expliqué dans le précédent article.

## Boucle de base

Le fichier **main.cpp** représentera pour l'instant l'application qui se servira de notre moteur. On appellera cette application **Little Launcher** !


### Initialisation de la librairie SDL

Dans la boucle, on commence par initialiser une librairie. La librairie **SDL** (la SDL3 pour être exact).

```C++
// main.cpp

int main(int argc, char **argv) {
    
    SDL_Init(0);    
```

La SDL nous sert notamment à abstraire des fonctionnalités relatives aux fenêtres, aux inputs et au rendu. Cela nous évite d'avoir à gérer les différents OS (ex : Win32, Posix) ou les différentes API graphiques (ex : DirectX, Vulkan, Metal)

Je vais m'appuyer sur elle pour le début du développement, et je la remplacerai petit à petit par mes propres implémentations plus tard.

On peut voir que je ne désinitialise pas la librairie à la fin de la boucle, c'est un oubli de ma part. Actuellement le programme ne libère pas proprement les ressources.

### Création de la fenêtre

Ensuite je crée une fenêtre en lui donnant un titre et des dimensions.

```C++
// main.cpp

    SDL_Window* window = SDL_CreateWindow("LittleLauncher", 1600, 900, 0); 
```

### Gestion des évènements

Enfin, on a la boucle infinie de l'application.

On parcours tous les évènements qui ont eu lieu entre la frame précédente et celle-ci. Si l'un d'entre eux est un évènement de fermeture de fenêtre ou de demande de fin de programme, on arrête le programme.


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

Ainsi, on a une petite application qui ouvre une fenêtre noire. Mais l'application n'est pas censée discuter avec la SDL directement.

Elle doit passer par le biais de notre moteur, et donc uniquement discuter avec les classes de notre moteur, qui vont venir faire une couche d'abstraction par-dessus la SDL.

## Module "Core"

Le module **Core** va accueillir nos premières classes, centrales au moteur. 
C'est son "noyau".

### Export

Chaque module contiendra son fichier d'export. Je vais montrer celui du module Core pour avoir un exemple.

```C++
// Export.hpp

#pragma once

#ifdef LE_CORE_COMPILE
#define LE_CORE_API __declspec(dllexport)
#else
#define LE_CORE_API __declspec(dllimport)
#endif
```

Ce petit fichier permet de définir deux macros que je placerai notamment au niveau de la déclaration de mes classes. Elles permettent de dire au compilateur deux choses :
- Lorsqu'il compile le module Core (avec la présence de LE_CORE_COMPILE), il doit marquer les symboles avec **__declspec(dllexport)** comme devant être exportés, pour pouvoir être utilisés plus tard par d'autres applications.
- Lorsqu'il compile une application qui utilise le module Core (LE_CORE_COMPILE est absent), **__declspec(dllimport)** permet de dire que les symboles proviennent d'une dll externe (celle de Core)

### Classe "Platform"

Commençons par *wrapper* l'initialisation et la finalisation de la librairie SDL.
J'ai créé une classe **Plateform**.

Le header est comme suit : 

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

Dans le fichier source je viens définir le constructeur et le destructeur :


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

Dans le constructeur, j'initialise la SDL. Je la finalise dans le destructeur.
L'avantage avec cette classe, c'est que maintenant je n'ai plus besoin de me rappeler que je dois appeler SDL_Quit.

```C++
//main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
```

Je crée une variable de type Platform, la SDL est initialisée. Une fois la fin du scope de la fonction main atteinte, le destructeur de "platform" est appelé et la SDL est finalisée. Parfait !

### Classe "Window"

La prochaine partie de la boucle à *wrapper* est la création de la fenêtre.

Je rajoute dans le module Core une classe **Window**.

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

J'ai rajouté le constructeur avec un titre et des dimensions, et j'ai retiré le constructeur par défaut.

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

Dans le constructeur, je crée la fenêtre SDL et je garde un pointeur vers celle-ci. Je m'assure de la détruire dans le destructeur pour libérer les ressources. Même principe qu'avec Platform.

On notera que j'avais aussi oublié, dans la boucle principale, de détruire la fenêtre. J'ai fait la boucle un peu trop vite et j'ai été tête en l'air.

Maintenant on remplace dans main.

```C++
// main.cpp

    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
```

### Les Évènements

Pour les évènements le principe reste le même. Comme les choses se répètent, je montrerai les headers seulement pour les définitions de certaines struct ou pour les énumérations.


D'abord la classe **Event** :

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
        // Et une quantité assez importante d'autres valeurs pour correspondre à tous les évènements de la SDL.
    }
```

Puis la classe **EventManager**. Le constructeur et destructeur sont gardés par défaut. On a juste une fonction PollEvent :

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
        // Au milieu, il y a un case pour chaque évènement SDL
        //...
        case SDL_EVENT_USER                              : event.eventType = EventType:: User                        ; break;
        case SDL_EVENT_LAST                              : event.eventType = EventType:: Last                        ; break;
    default:
        throw std::runtime_error("EventManager : Unhandled event type");
    }

    return true;
}
```

Finalement, on remplace dans la boucle principale et voici le résultat final :

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

C'est super ! Maintenant on affiche toujours une fenêtre noire, mais avec nos propres classes hehe !!

Il y a une chose dont je ne suis pas vraiment satisfaite. Dans le header de la classe Window, on a un pointeur vers une SDL_Window et on fait la déclaration anticipée de ce type. 
Le problème pour moi avec ça, c'est que techniquement, l'application qui parle avec le module Core voit encore la SDL à travers lui, alors qu'elle ne devrait jamais voir le bout du nez de la SDL.
On règlera ce souci plus tard.

## Dessiner nos premières formes

Pour pouvoir afficher quelque chose à l'écran, on va utiliser le **SDL_Renderer**.

### Création du Renderer

Pour l'instant, on va le créer dans le **main.cpp** pour tester.

{{< image src="little-engine/devlog/first-shapes/creation-SDLRenderer.png">}}

On s'aperçoit que l'on a besoin d'un pointeur vers une SDL_Window pour créer le renderer, mais notre classe Window n'a pas de fonction pour cela. Ajoutons une fonction *getter* publique.

```C++
// Window.cpp

SDL_Window* Le::Window::GetHandle()
{
    return m_handle;
}
```

On a maintenant le code suivant pour créer le renderer

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
*J'utilise **fmt** pour logger dans la console.*
*C'est une librairie pratique, notamment pour le formatage de chaîne de caractères.*
{{< /alert >}}

Cependant, si j'essaie de lancer le programme, la console affiche le message lorsque **renderer** n'est pas valide.

{{< image src="little-engine/devlog/first-shapes/creation-renderer-error.png">}}

Cette erreur m'a surprise quelques secondes. Jusqu'ici, j'étais plutôt en terrain connu. Je reproduisais ce que j'avais fait durant mon cours de moteur. La seule différence est que dans mon projet de cours, je liais la SDL dynamiquement (.SDL3.dll), alors qu'ici je le fais statiquement (SDL3.lib).
Et là est le problème. Voici ce que j'ai cru en comprendre.

### Liaison statique à la SDL

Dans mon cours de moteur, j'avais décidé de faire quelque chose comme ça : 

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->C[SDL3.dll];
B[Core.dll]-->C
{{< /mermaid >}}

Les deux binaires se liaient à la même instance de la SDL car j'utilisais la version dll. Donc même si j'initialisais la SDL dans le binaire du module Core, celui de LittleLauncher parlait avec la même instance, déjà initialisée. Donc pas de problème.

Mais cette fois-ci, j'ai voulu lier la SDL de manière statique pour ne pas me trimballer la dll, et que le code de la librairie fasse partie intégrante de mon code source.

Sauf que pour le test du renderer, le module Core n'est pas le seul à se lier avec la SDL. Little Launcher le fait également pour créer un renderer. Mais comme la SDL est liée de manière statique, chacun binaire parle à une instance différente de la SDL.

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->C[SDL3.lib];
B[Core.dll]-->D[SDL3.lib]
{{< /mermaid >}}

Core initialise son instance de la SDL. Mais celle de LittleLauncher reste non initialisée.

Et j'avais complètement oublié ce détail.

**Il faut que tous les appels à la SDL se fassent depuis le même module !!**

Cependant, je ne veux pas juste créer une classe **Renderer** dans le module Core. J'avais plutôt en tête de faire du rendu un module différent. Le module **Rendering**.

Et on se retrouverait avec le même problème.

{{< mermaid >}}
graph TD;
A[Rendering.dll]-->C[SDL3.lib];
B[Core.dll]-->D[SDL3.lib]
{{< /mermaid >}}

## Module "Platform"

Pour régler ce problème, je me suis dit que j'allais créer un module qui serait responsable de la discussion avec la SDL. Tous les appels à cette dernière ce feraient depuis ce dernier. Le module **Platform**.

Actuellement on a ce graphe de relation entre les binaires : 

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Core.dll];
{{< /mermaid >}}

Si on ajoute le module Platform, et que l'on déplace tous les appels vers la SDL à l'intérieur, on se retrouverait avec ça :

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Core.dll];
A[LittleLauncher.exe]-->C[Platform.dll];
B[Core.dll]-->C[Platform.dll];
{{< /mermaid >}}

Alors c'est parti. Pour la classe Platform, c'est simple, je la déplace dans le module Platform. Pour les autres classes, c'est un peu différent...

### WindowImpl

Je vais créer une classe **WindowImpl** qui est une copie conforme de la classe Window. WindowImpl sera dans le module Platform.

{{< alert "circle-info">}}
*Je ne montre pas le header et le cpp. C'est une copie identique de window avec un nom de classe différent.*
{{< /alert >}}

Ensuite je modifie la classe Window pour ne pas avoir une handle vers une SDL_Window directement, mais vers une WindowImpl.

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

Je vous remontre le header pour spécifier quelque chose. Plus haut j'avais parlé du fait que je n'aimais pas le fait que l'application voyait la SDL à travers le header de Window.

Maintenant, la communication avec la SDL se fait dans WindowImpl, et je profite de l'occasion pour abstraire complètement l'implémentation dans Window. On ne voit plus aucun type de la SDL dans le header de Window. Le header de WindowImpl est inclus dans le .cpp uniquement.

Bien sûr, il n'y a rien de révolutionnaire dans ce pattern. On fait une déclaration anticipée de WindowImpl pour ne pas inclure le header dans le .hpp et potentiellement alléger le temps de compilation.

Mais j'étais content qu'un effet secondaire de cette modification soit une meilleure abstraction de la SDL aux yeux de l'application qui utilise mon moteur.

Bref, revenons en à nos moutons.

Voici le .cpp de Window maintenant : 

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

J'en ai profité pour ajouter le constructeur et l'assignation par mouvement, ainsi qu'un *getter* pour **m_impl**. Je me suis dit que ça pourrait être utile.

### Déplacement des classes liées aux évènements

Je déplace la classe Event dans le module Platform.

Et pour l'EventManager, je fais le même exercice qu'avec Window en créant dans le module Platform une classe EventManagerImpl. Le principe est le même.

### RendererImpl

Maintenant, on a toujours la même boucle, mais avec tous les appels vers SDL faits depuis le module Platform.

On va faire la même chose avec le SDL_Renderer et faire une classe RendererImpl dans le module Platform.

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

Pour l'instant, on va appeler ça directement dans la fonction main.


```C++
// main.cpp

int main(int argc, char **argv) {
    
    Le::Platform platform{};
    Le::Window window("Little Launcher", 1600, 900);
    Le::EventManager eventManager{};
    
    Le::RendererImpl renderer(*window.GetImpl());
```

Maintenant, si on lance l'application, ça marche !

## De retour sur le rendu des formes

Le problème est réglé, on peut enfin essayer d'afficher nos premières formes.

Je rajoute quelques fonctions à RendererImpl pour faire un rendu simple

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

Et voici main :

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

Petite image du résultat !!

{{< image src="little-engine/devlog/first-shapes/shapes.png" >}}

On peut passer à la dernière étape de la semaine : la création du module **Rendering**

## Module Rendering

Je crée un module Rendering et une classe Renderer qui va dans ce module.

Ensuite je fais le même exercice qu'avec Window. Renderer devient un *wrapper* de RendererImpl.

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

Dans main, je remplace la variable de type RendererImpl par le type Renderer et fais les ajustements nécessaires.

J'en profite pour utiliser le concept de **"friend class"** pour rendre la fonction **Window::GetImpl** privée. En théorie le Renderer est le seul à avoir besoin de cette fonction. De plus, l'application n'est pas censée avoir accès à WindowImpl, car cela lui permettrait alors indirectement de voir les types de la SDL, ce dont je n'ai pas envie.

Avec le module Rendering, on a les relations suivantes :

{{< mermaid >}}
graph TD;
A[LittleLauncher.exe]-->B[Rendering.dll];
B[Rendering.dll]-->C[Core.dll];
A[LittleLauncher.exe]-->C[Core.dll];
B[Rendering.dll]-->D[Platform.dll];
C[Core.dll]-->D[Platform.dll];
{{< /mermaid >}}

## Conclusion

C'est tout pour cette semaine !

Si vous avez des recommandations et conseils sur comment améliorer mon devlog ou mon moteur, n'hésitez pas à me contacter :
* Sur **[LinkedIn](https://www.linkedin.com/in/antoine-hanna/)** 
* Par mail : {{< email email="mailto:ahanna.pro@gmail.com" text="ahanna.pro@gmail.com" subject="">}}

Vous pouvez tout retrouver sur le repo du projet : 
{{< github repo="Haiito92/LittleEngine" showThumbnail=true >}}

Je vous souhaite une bonne journée/soirée et à la semaine prochaine !