---
layout: post
title:  "VS Code add bower build tasks to default build task"
date:   2017-12-13 00:00:00 -0500
categories: programming
---

The .NET Core CLI gives you some useful templates for quickly newing up projects and getting going.

One of them is:

```
dotnet new mvc
```

Which sets you up to use bower for package management of frontend-y stuff.

One of the things that the template does not set you up with (and rightly so) is a way to automatically
apply your bower config on every build.

This can be done manually in the terminal by typing the following any time you make a change to your Bower
configuration:

```
bower prune
bower install
```

But we are lazier than that and just want it to be taken care of on build. One way to achieve this is to create
some tasks in our .vscode/tasks.json file that run these commands.

We can then chain them together with dotnet build into one task which acts as our default build task in VS Code.

That way when we press CTRL + B, we are running all three of these commands together.

Here is a sample tasks.json that shows how to achieve this. If you are new to using .NET Core and VS Code, these
tasks run on the folder level which is typically going to be the root of your repo and where you .sln lives.
For the bower tasks you will need to set your current working directory for each relevant web project in your solution.

```
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "dependsOn": ["bower-prune", "bower-install", "dotnet-build"],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "label": "dotnet-build",            
            "command": "dotnet build",
            "type": "shell",
            "group": "build",
            "presentation": {
                "reveal": "always"
            },
            "problemMatcher": "$msCompile"
        },
        {
            "label": "bower-prune",
            "options": {
                "cwd": "${workspaceFolder}/src/yourproject.web"
            },
            "command": "bower prune",
            "type": "shell",
            "presentation": {
                "reveal": "always"
            },
            "group": "build"
        },
        {
            "label": "bower-install",
            "options": {
                "cwd": "${workspaceFolder}/src/project.web"
            },
            "command": "bower install",
            "type": "shell",
            "presentation": {
                "reveal": "always"
            },
            "group": "build"
        }
    ]
}
```