---
title: Discovering XWorm V5.4 in malicious Github repository 
date: 2024-06-26 22:30:00 +0200
categories: [malware, xworm]
tags: [malware xworm worm obfuscated github]     # TAG names should always be lowercase
---

# Stage 0 - Finding the suspicous github repository

Last week, I was just browsing through YouTube and found a very interesting [video](https://www.youtube.com/watch?v=yjLYz2lo0FE), by Eric Parker, about a Discord info stealer written in Python. In this video, he shows the PySilon malware and analyzes what happens after you get infected with it. Halfway through the video, I saw that the malware was actually open-source, and I just had to look into the [GitHub repository](https://github.com/mategol/PySilon-malware). After browsing the code for a bit, I realized that there were a lot of open issues present in the repo and thought, "Who TF would open an issue on a malicious repository? This would only be the case if, and only if, you wanted to use the malware yourself! Right?". But who has problems running a Python script and then opening an issue because of that?
Seems like, the majority of people have no idea, in the slightest, how a programming language works. I would go so far and say, these kids..., I mean, grown-up adults who definitely have a computer science degree, create a GitHub account just for the sake of asking why the program exits with the error ["'python' is not recognized as an internal or external command, operable program or batch file."](https://github.com/mategol/PySilon-malware/issues/389).
But just for a moment, let's not talk about script kiddies who are having problems with running some malware, because I know we were all there once 😬, and rather talk about the sketchy profiles who wrote these issues in the first place, like the one of [zen1em](https://github.com/zen1em). Maybe take a look at it for yourself, because there are a bunch of weirdly named repos on their GitHub account. For example [RoPrebfdgrs](https://github.com/zen1em/RoPrebfdgrs), which immediately caught my eye.
And that brings us to...

# Stage 1 - The Batch loader 

The repository just contains the RoPredictorV1.7.zip file. 
After unzipping, you get an obfuscated batch file with a huge chunk of Base64-encoded data at the bottom.
Let's call that the ``Base64 blob`` for now.
At first sight, this file is unreadable, and therefore, I did some deobfuscation, and this is the result.

![Deobfuscated batch file](/assets/RoPrebfdgrs/img/RoPredictorV1.7%20beautified.svg)

The most important task of this program is that it reads itself and splits its content on every new line, storing it in the `$RoPredictorSplitNew` variable.
As seen in the original file, there is a huge blob of Base64-encoded data at the bottom of the original file. 
This is decoded, stored again, and iterated over each character, performing a xor operation with some kind of key, the `$Base64Key`.
The last step of this first file is decompressing the new decoded Base64 blob, with GZip, and loading the executable via the `[System.Reflection.Assembly]` call, and we are left with...

# Stage 2 - A .net executable  

This .net assembly serves the same purpose as the Batch loader, it's just written in C#.
Additionally, it ships with two resources: 
- payload.exe
- apiunhooker.dll

After some deobfuscation, it's clear that the program only uses three functions to load the final payload.

At first, it uses the `GetManifestResourceStream()` function to access the `payload.exe` resource. 

![GetManifestResourceStream](/assets/RoPrebfdgrs/img/GetManifestResourceStream.svg)

Secondly, it uses `DecryptInput()` to decrypt the encrypted payload.

![DecryptInput](/assets/RoPrebfdgrs/img/DecryptInput.svg)

And lastly, it decompresses the previously decrypted data with the `DecompressGZip()` function.

![DecompressGZip](/assets/RoPrebfdgrs/img/DecompressGZip.svg)

There is also some more functionality in the main function, but I hadn't had the time to look into it. 
Now, the .net executable calls to `[System.Reflection.Assembly]` again, running the `payload.exe` file as a result. Finally, this leads us to...

# Stage 3 - Final Payload

Here is the file tree of the decompiled `payload.exe` in ILSpy.

![filetree_0](/assets/RoPrebfdgrs/img/payload_filetree_0.png)

As you can see, this file is way harder to analyze than the first .net executable we looked at in the previous section.
For example, here are the functions exposed by `Stub.9BBWUUOv3LzC495FCq9MTz0JmlS9qEa3zQACsrVYTQUOl0uBy3YCOFkVUAzt`, just for you to see how bad it really is.

![filetree_1](/assets/RoPrebfdgrs/img/payload_filetree_1.png)

And thats just one of many classes and namespaces.
By the time of writing, I have just looked into the main function of the payload. But nevertheless, there were some interesting strings in there. 
This is the main function that gets executed by the first .net executable.

![Main function strings](/assets/RoPrebfdgrs/img/payload_main_strings.png)

After a lot of deobfuscation, searching for functions and decrypting, I was greeted by some nice strings.

![unobfuscated_payload_main](/assets/RoPrebfdgrs/img/unobfuscated_payload_main.png)

It's clear that we are dealing with XWorm version 5.4 here. Just by looking through the other files and functions, there is a lot more to discover in this final payload.
For example, there is a huge switch statement, which is probably there for incoming commands. The commands I found are: 

- "pong"
- "rec"
- "CLOSE"
- "uninstall"
- "update"
- "DW"
- "FM"
- "LN"
- "Urlopen"
- "Urlhide"
- "PCShutdown"
- "PCRestart"
- "PCLogoff"
- "RunShell"
- "StartDDos"
- "StopDDos"
- "StartReport"
- "StopReport"
- "Xchat"
- "Hosts"
- "Shosts"
- "DDos"
- "ngrok"
- "plugin"
- "savePlugin"
- "RemovePlugins"
- "OfflineGet"
- "$Cap"

Although, I do not think these are all commands that can be executed by an attacker, it would be quite interesting to look at just these "base functionalities".
There are also references to load plugins and other files from the web, but I won't cover any more features here in this post, but I'm planning to do so in another. 
And lastly, if you are interested in any of the files, they are all in [the GitHub repository](/assets/RoPrebfdgrs/files/RoPredictor_(X-Worm)_PW_"infected".zip).
