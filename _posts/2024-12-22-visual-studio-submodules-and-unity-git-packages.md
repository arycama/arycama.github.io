---
layout: post
title: Visual Studio Submodules and Unity Git Packages
---

## Visual Studio Submodules and Unity Git Packages

I've been using Unity git packages to develop a few different libraries alongside my game projects, such as my Custom Render Pipeline. (https://github.com/arycama/customrenderpipeline) While the setup isn't too complicated, and Unity will automatically pull it into the project/update when needed, developing the pipeline while also developing my project can be a bit tedious, as I'll need to switch back and forth between different visual studio projects to make commits to either one. I also need to ensure that the changes to my package aren't being tracked/included in my main project, and vice versa. 

Thankfully Visual Studio has added support for managing multiple submodules from a single Visual Studio instance. However, it doesn't quite work out of the box with Unity thanks to the way Unity generates it's files. (And gives us no control over)

First, we need a Unity git package. There are tutorials online for setting up a custom git package for Unity, but I'll cover the basics. First, you'll need to create a github repo. Check this out into the packages directory in your project, and into a subfolder with the format com.<company-name>.<package-name> eg com.arycama.customrenderpipeline. In this folder, create a package.json file, and add some basic details. Check this file for an example: https://github.com/arycama/customrenderpipeline/blob/master/package.json

Open Unity and it should import this package, and you'll see it under your packages directory. You can now add and modify files. (I'll talk more about the solution/Git.csproj files you see below in a minute)

![image](https://github.com/user-attachments/assets/eed503a0-ee7d-4e09-80b1-d0d0add80fa4)

You will also want to enable .csproj file generation for Git Packages under Preferences/External Tools, so that your package code can be edited and browsed in Visual Studio.

![image](https://github.com/user-attachments/assets/056b17fb-593a-4ba6-8139-f521412bd7fb)

Once you've added some code etc to your package, you'll want to commit it. However you want to commit it to your package repo, not your main Unity project repo. If you're using Visual Studio for version control, you can open the package folder directly, and it should detect the git repo, and you can commit as usual. However, having to switch back and forth constantly can be tedious. You'll also have to do this to update the package if external changes are made, or you'll have to update the package through Unity's package manager. Wouldn't it be great if you could just update your submodules/packages at the same time as your main project? Read on.

In Visual Studio, create a new blank solution by File/Project/New Project and select "Blank Solution". Create this in your package folder, eg com.arycama.customrenderpipeline, and call it the name of your package, in my case, "CustomRenderPipeline". Visual studio annoyingly creates this in a subfolder, so quit visual studio and move the solution up to the main package folder.

Now, re-open the solution. In Visual Studio's solution explorer, right click the empty solution, click Add, and select "Existing Project". Now, navigate to your Unity project's root directory, and you'll find an existing solution for your git project already exists, thanks to the git project generation option we enabled above. Add this to your solution. This will work regardless of Unity project/solution regenerations, as it will be stored using relative paths.

Now, there is one more slightly strange step. We need an actual .csproj file in this package directory, or Visual Studio won't detect this as a git submodule when the main project is open. I simply created an empty .csproj in this directory with the same name as my package, but with a .Git suffix, eg CustomRenderPipeline.Git. 

So, now we need the main Unity project's solution to reference this, so it gets detected as a submodule. We can manually add it as an existing project in Visual Studio. However, Unity will regenerate this solution for various reasons, so our changes will be lost. Luckily, there is a way to hook into Unity's solution generation process. If we make a class that derives from AssetPostprocessor (Make sure to put it in an Editor folder/assembly) and add a static void called "OnGeneratedCSProjectFiles", it will get called whenever the solution gets regenerated. 

From here, we can simply load the solution file as text, and append the projects we need. While it may be possible to iterate over all of the project's git packages and write out this data programatically, in my case, I only have four projects I want to include as submodules and this likely won't change often, so I'm just going to hardcode them. This file can be committed to your main projects version control and it should work for anyone else on the team. If you do find a nicer way to automate this though, let me know!

Note that I do a check to make sure the existing text does not already exist. Unity will sometimes call this callback multiple times, so you will get the project added multiple times which will make Visual Studio annoyingly unhappy.

```csharp
using UnityEngine;
using UnityEditor;
using System;
using System.IO;

public class PostProcessVisualStudioCSProject : AssetPostprocessor
{
    static public void OnGeneratedCSProjectFiles()
    {
        var projectDirectory = Directory.GetParent(Application.dataPath).FullName; ;
        var projectName = Path.GetFileName(projectDirectory);
        var slnFile = Path.Combine(projectDirectory, $"{projectName}.sln");
        var slnText = File.ReadAllText(slnFile);

        using (var sw = File.AppendText(slnFile))
        {
            void WriteProject(string name, string path)
            {
                if (!slnText.Contains($"{name}.Git"))
                {
                    var guid = Guid.NewGuid();
                    var projTypeGuid = "FAE04EC0-301F-11D3-BF4B-00C04F79EFBC";
                    sw.WriteLine($@"Project(""{projTypeGuid}"") = ""{name}.Git"", ""{path}\{name}.Git.csproj"", ""{{{guid}}}""");
                    sw.WriteLine("EndProject");
                }
            }

            WriteProject("CustomRenderPipeline", "Packages/com.arycama.customrenderpipeline");
            WriteProject("NodeGraph", "Packages/com.arycama.nodegraph");
            WriteProject("TerrainGraph", "Packages/com.arycama.terraingraph");
            WriteProject("WebGLNoiseUnity", "Packages/com.arycama.webglnoiseunity");
        }

        Debug.Log("Appended package projects to solution");
    }
}
```

Now in your visual studio window, you will notice it is tracking multiple repositories as submodules. You can now change files in any of them, and commit them from the same window that you use for regular git commits, and the files will be committed to their respective repos. 

![image](https://github.com/user-attachments/assets/42dc40b1-46a6-4a23-bc4d-55e827e63790)

Happy sub-module-ing!
