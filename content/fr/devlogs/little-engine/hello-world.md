+++
date = '2026-10-02T17:35:54-04:00'
draft = false
title = 'Hello World'
showWordCount = false
+++

## Introduction

**"Hello World!"**. Ceci est le tout premier post de mon devlog pour Little Engine.

Little Engine est, pour moi, un projet avant tout pour le fun, parce que j'aime bien rentrer dans le code de bas niveau. 
C'est par la même occasion un projet d'apprentissage et de découverte.

L'idée c'est d'abord de faire un petit moteur en m'aidant des librairies communes.
SDL, Box2D, Jolt, etc...

Puis, un par un, remplacer les middlewares/librairies par mes propres intégrations faites de zéro !

Je compte poster à un rythme d'un post par semaine. Parfois les avancées seront grandes, parfois petites. Tout sera partagé avec vous !

## Première étape : Hello world

Pour cette semaine, j'ai simplement fait un petit bout de code, en utilisant **SDL3**, pour ouvrir une fênetre que l'on peut fermer.

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

## Configuration Xmake

Le code engine de cette semaine se résume au petit script du dessus.
Le reste de mon travail cette semaine était surtout un peu de préparation de la config **Xmake**.

Xmake est un **outil de build**. Il permet de créer des fichiers de configuration qui facilitent notamment: 
* La generation de projet
* La gestion des packages
* La gestion de la compilation
* La build/le packaging


J'ai mis en place le xmake.lua (le fichier config principal), ainsi que deux fichiers "task" pour faciliter la création de module et de classe.

### Fichier de config : xmake.lua

Le fichier de config permet de gérer la compilation, la séparation des modules, les targets, etc...

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

### Tasks xmake

Les fichiers "tasks" me permettent de créer mes propros commandes xmake pour réaliser des tâches diverses.

Je peux créer un fichier .lua (que j'importe ensuite dans le fichier de config principal), et déclarer une tâche. Voici en exemple la tâche de création de classe :

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

Via set_menu, je dis à xmake comment appeler la tâche.
Je peux ensuite l'éxécuter dans l'invite de commande.

```pwsh
xmake create-class -n Application -m Core
```

## Conclusion

C'est tout pour cette semaine !

Si vous avez des recommandations et conseils sur comment améliorer mon devlog ou mon moteur, n'hésitez pas à me contacter :
* Sur **[LinkedIn](https://www.linkedin.com/in/antoine-hanna/)** 
* Par mail : **ahanna.pro@gmail.com**

Vous pouvez tout retrouver sur le repo du projet : 
{{< github repo="Haiito92/LittleEngine" showThumbnail=true >}}

Je vous souhaite une bonne journée/soirée et à la semaine prochaine !

