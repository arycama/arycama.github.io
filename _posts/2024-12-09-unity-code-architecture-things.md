---
layout: post
title: Unity Dependencies, Singletons, Scene Loading, Factories.. etc
---

## Unity Dependencies, Singletons, Scene Loading, Factories.. etc

To manage dependencies, I use additive scene loading and a DependencyResolver component. The first scene to load is the "Global" scene. This will never be unloaded, and contains any systems you want to persist for your entire game. In a basic example, the Global scene contains a "Music Player", a "Global Dependency Resolver", a "Scene Loader" and a "Player Input" GameObject. Each one of these has a respective component. (Or two, eg in the case of Music Player, it also has an AudioSource) I prefer a flexible hierachy of different GameObjects, and usually one per system, instead of one bloated GameManager component or object. The GlobalDependencyResolver is quite simple, it is a list of references to components on objects, in this case, it contains a reference to musicPlayer, SceneLoader, and Player Input.

!(/assets/img0.png)

This is basically saying "This is the global music player, global scene loader, global player input". (As they are all in the global scene) You'll note that these can be easily assigned in the inspector, you can add new global components and assign them easily. I'll go into how this is used by scripts soon.

The DependencyResolver is fairly straightforward, it first checks to see if it's current scene already contains a parent resolver. This might seem strange, but I mentioned I have a hiearachy of scenes and dependencies. This allows a "child" scene to get the Music Player without having to explicitly specify it. It will simply look at the parent resolver if it does not contain a direct reference. This also solves the problem of the Global scene not having to know about all the component types in other scenes. (Extensibility? Oh my)

!(/assets/img1.png)

It simply adds each component to a Dictionary, keyed by type. You'll note that it is keyed by scene. Internally, each scene has an int as a handle, which is used as it's hash code, so this is quite straightforward  computation-wise. 

We have a single static Dictionary that maps from scene (handle) to a DependencyResolver. This is the only static part of the system, as the whole point of dependencies is to allow for containerisation and encapsulation, and having globally available systems somewhat breaks that.

!(/assets/img2.png)

So, the basic idea is.. The global scene loads, depenedncies are assigned, and then it can now load another scene. Anything this child scene needs as a dependency will then get setup by it's own dependency manager (If present) and if no dependency manager is present for the scene, it will attempt to load it's parent resolver instead. (Which will be the global scene) Multiple levels of inheritance are possible.

So, now we want to load our "game" scene. This contains systems that will generally be persistent throughout the actual gameplay, eg after entering your main menu, and clicking "new game/load game" etc, it will load these things. This will include a system that loads the actual level (Which can be it's own scene, a child of the Game scene, with it's own dependencies, and can also inherit dependenceis from the Game scene (And the global scene by extension)).

The game scene's resolver looks like so :

!(/assets/img3.png)

Nothing too complicated, a general Event System and Canvas for UI, as well as a Player Factory and Mission Controller. Once nice thing about a hierachical scene/system setup is that I don't need to worry too much about cleanup code. The scene can simply be reloaded which will unload everything, and then reloaded which will re-initialize everything, as well as dependencies. Global dependencies (Eg from the global scene) however will remain persistent, as expected.

A note about scene loading, it's important that the dependency for a scene is set before it finishes loading. This can be achieved by callign Scenemanager.LoadSceneAsync, and then immediately afterwards, resolving the scene. Unity scene loads always take at least one frame to complete, so this is safe.

Now about that PlayerFactory, what does it do? It spawns the player of course, but why is this necessary? Well, various components need to be hooked up to the player. For example, UI prefabs displaying text, health, ammo count, a crosshair, etc. I like to keep my UI code seperated from gameplay, so that game code does not have knowledge of UI, but simply exposes read-only variables, such as current ammo etc. So our player factory can create the player, get its weapon component, and then create an AmmoText prefab, passing that weapon as an argument. 

PlayerFactory.SpawnPlayer:
!(/assets/img4.png)

I use static "create" methods as a way of assigning read-only variables on creation, similar to a constructor, which we know does not work well with MonoBehaviours. I choose not to have a monolithic UI prefab in my game at all times, instead I have modular prefabs that can be created/assigned to as needed. This solves a lot of management and versioning problems that come with configuring/maintaining large prefabs in editor.

!(/assets/img5.png)

As I add more logic and complexity to my player spawning, I can simply add the extra prefabs/dependencies to PlayerFactory and spawn things as needed.

I also have a MissionController that you may have seen earlier. This could take on other names such as quest manager, game controller, etc.. This simply tracks an active mission (Or missions), updates it, checks for completion, and if so, removes it from the list and invokves its on completion function.

!(/assets/img6.png)

A mission is made up of a ScriptableObject called "missionData" which contains the objectives, such as collect 5 wood. A second class called an ActiveMission is created at runtime, and this tracks the state of the mission and has the logic. These derive from an abstract mission calss with a simple constructor and Update function. (But no end function, since an Update function returns a true/false bool, indicationg completion. Any completion logic can be done here.

More to come. (And better examples, sample project etc)
