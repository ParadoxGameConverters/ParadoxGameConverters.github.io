## Introduction - Normal Debugging ##

It began with a user report of the [Vic2 To HoI4 converter](https://steamcommunity.com/sharedfiles/filedetails/?id=733122837) crashing. Sadly, this is not uncommon, but it's fairly infrequent these days.

Typically debugging is pretty dependent on the input data, namely a Victoria 2 save and possibly a Vic2 mod to go with it. I asked the user for theirs and they obliged.

I tried it on my machine and sure enough, got a hard crash. At least it wasn't a problem only on the user's machine.

The next step was to run it in Visual Studio and see where the crash happened. When it happened, it was [in a library function, freeing a manually-allocated array](https://github.com/ParadoxGameConverters/commonItems/blob/759705dc41ff1595db9786d0a9facc0b336888c3/targa.cpp#L1158). Note, I debug using a release build because that makes it easier to set up the necessary run-time config. This step is easier in a debug build and may make some of the following steps unnecessary.

Knowing where the crash happens, I added ```#pragma optimize("",off)``` before the function and ```#pragma optimize("",on)``` after. This turns off debugging just for that stretch of code, allowing me to use the debugger to look at the local variables. On doing so, it was clear the freed variable wasn't null, and in fact pointed to valid-looking memory.

So next I looked at the call stack to see who was calling the library function. It turned out to be [HoI4::processFlagsForCountry()](https://github.com/ParadoxGameConverters/Vic2ToHoI4/blob/0b6af98a13ae54ffb32af9bff1d9cc1037a1ef56/Vic2ToHoI4/Source/OutHoi4/OutFlags.cpp#L100-L146), a function that takes a flag from Vic2 (formatted as a .tga file), generates flags for HoI4 from it, and then releases the flag.

After a repeat of the trick to debug, I was able to see that it was specifically a generated dominion flag. Generated domains are countries that are created from scratch, including flags that are completely constructed in memory from a base color, another flag (added [in canton](https://en.wikipedia.org/wiki/Canton_(flag))), and an emblem. Both code inspection and watching a run under the debugger showed nothing strange happening in the creation of the flag.

At this point I ran all the way to the crash again and looked more closely at the error. It mentioned heap corruption, which I understood to mean I was writing outside the allocated memory at some point. A quick Google search [suggested to use pageheap to track where this happens](https://docs.microsoft.com/en-us/visualstudio/debugger/how-can-i-find-out-if-my-pointers-corrupt-a-memory-address-q?view=vs-2019). Unfortunately, all the information I could find assumed familiarity with debugging from the command-line, as this tool is not incorporated into Visual Studio.

## Running pageheap ##
It was time to learn. The first difficulty was figuring out how to get it. After some further research, I found that you need to install the [Windows 10 SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/). That was pretty straightforward.

But it didn't seem to work when I tried to run the commands.  After more research, I found that the tools all install to ```C:\Program Files (x86)\Windows Kits\10\Debuggers\x64``` and don't get added to path.

One more note - they require administrator permissions to work, but don't obviously fail when run without these permissions.

[This page](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug) actually turned out to be a decent guide, once I was able to find it.

So my process to run pageheap with my tools was:  
* Press the start button  
* Type ```cmd```
* Right-click on Command Prompt and choose 'Run as administrator'
* Navigate to the directory where I have my executable to investigate: ```cd "C:\MyProjects\paradoxGameConvertersForks\Vic2ToHoI4\Release\Vic2ToHoI4"```
* Enable gflags for the binary: ```"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\gflags" /p /enable V2ToHoI4Converter.exe```

At that point, I could run ```"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\gflags" /p``` and V2ToHoI4Converter.exe was listed, so I knew it was enabled.

Finally, it was time to run my program, but making sure to do so under the debugger: ```"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\ntsd" -g -x V2ToHoI4Converter```

At that point my program was running, but extremely slowly. I took the time to check the math on my functions that are doing all the pointer arithmetic and determined that for the constraints on my inputs they couldn't be going over. It was quite the mystery.

Finally, the debugger stopped with an error, but it wasn't clear where. But from the page listed before, these two command where the thing to do:  
```.lines```  
```kb```  

And with that, a stack trace dropped out, showing the exact line where I was writing out of bounds.

```q``` exited the debugger, and ```"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\gflags" /p /disable V2ToHoI4Converter.exe``` disabled pageheap on my program.

## Fixing the bug ##
Where I had expected errors to be while copying emblems, as that function had more complex math, the error was actually while [copying the parent flag in canton](https://github.com/ParadoxGameConverters/Vic2ToHoI4/blob/2f550b059501fbc556c707745a49eb3b60efb8f3/Vic2ToHoI4/Source/OutHoi4/OutFlags.cpp#L293). It was an easy matter to add some debug code to stop when this condition happened. As most of you probably predicted, one of the flags violated the size constraints. While that one flag would have been easy to fix, the converter has nearly seven thousand flags to potentially review, and it can read flags from mods or accept user-provided flags. Ensuring the constraints were followed was clearly impossible. So instead I did what I should have done when first writing the code and [put protections in place](https://github.com/ParadoxGameConverters/Vic2ToHoI4/commit/9b494abaacdd37140b312db4a3cc1f54572dc2dd). Running the converter again demonstrated that the crash was fixed. After allowing the automated deployment to run, I instructed the original user to download the new version and they reported success.