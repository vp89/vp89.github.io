---
layout: post
title:  "VS Code lldb debugging with stdin"
date:   2018-04-03 00:00:00 -0500
categories: programming
---

I've been trying to get away from these small "how to" posts, but it took me a while to find a solution
for this so it feels worthwhile to do a quick write up on it.

I was struggling to pipe in stdin to some C programs I wanted to debug using lldb from VS Code.
After much Googling I finally found a solution and wanted to post a complete example.

The below is a stock lldb debug task config (generated by VS Code) that should go in your .vscode/launch.json file.

I made two changes:

1) Add a property called `setupCommands`.

This allows you to feed lldb commands in used to setup the debugger. You will need to modify where it says `<pathToYourFile>`
with a path relative to the `cwd` defined in this same configuration.

2) Remove the `"externalConsole": true` line

{% highlight json %}
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(lldb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "enter program name, for example ${workspaceFolder}/a.out",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "MIMode": "lldb",
            "setupCommands": [
                {
                    "text": "settings set target.input-path <pathToYourFile>"
                }
            ]
        }
    ],
    "compounds": []
}
{% endhighlight %}

I tried piping in some input through the `args` property but it wasn't reading it. 
If anyone has a better solution feel free to share, happy debugging.