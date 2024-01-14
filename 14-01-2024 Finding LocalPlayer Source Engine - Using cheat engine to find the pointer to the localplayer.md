## Introduction

Figured I'd make some use of this site, this is a little tutorial on how to use Cheat Engine to find a static pointer to the localplayer in Source Engine games, i'm using Garry's Mod for this.

## sauce

First, of course, attach cheat engine to your game.

When scanning for values in Cheat Engine you should use the correct value type, for this, we're looking for health and in Source Engine it happens to be stored as an int which has a size of 4 bytes, so the `Value Type` should be 4 bytes.\
Open up the game and join a dedicated server. You'll need to use an initial value, I assume there's probably more 100's than there are 63's so I'm going to get my health down to that just incase it's faster - might not be though, but it's cool to save 20 miliseconds when scanning.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/80164fa5-5e8a-4171-bd29-628fc7127129)


Keep adjusting your health, increasing, decreasing, `Next Scan`ing in cheat engine every time until you have as few as you can get, manually remove the flickering values if there are any as that wont be your health, of course.\
Took me 2 scans, one at 63hp, the second when I was at 51hp.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/787acc9b-f108-4bac-9583-58a5e0ed0e08)

Looking at the addresses, I'm most intrigued by the two bottom ones as they're the furthest away from the other's. The top ones are all in the `0x2228B`-`0x2228F` region, whereas the bottom two are `0x222A1`. \
Going to start with my `222A12DC0C8`, right clicking then select `Found out what accesses this address` (or press f5 shortcut). \
![image](https://github.com/RegiSimkus/blogs/assets/91128330/d76205ab-e10a-427c-8651-e4fa51c9b84e)
Here I see these following instructions accessing this health address, you can see `[rcx+C8]` occuring often which tells us that `0xC8` is the offset for health in the player structure (can change when the game is recompiled/gets updates).\
You can click on one of them, whatever looks most interesting to you, I like the look of `cmp dword ptr [rcx+C8],00` as that's just a check to see whether we're dead.\
Clicking on the instruction, you can see the values in the registers at the time in the bottom part of cheat engine, take the value of rcx and add it to the address list.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/c7096af2-4246-4c7e-9e3f-c6a23f0d865a)

Take that pointer and scan for it to see what addresses point to the localplayer. 8 bytes because i'm on a x64 version of the game where pointers are 8 bytes large, whereas x86 would be 4 bytes.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/81598616-929a-4c28-8162-e659ec25468f)

Lots of garbage here at first glance
![image](https://github.com/RegiSimkus/blogs/assets/91128330/0b191538-a9a5-4e52-b3df-7e97a1ada086)
Scroll down a bit and wey hey!
![image](https://github.com/RegiSimkus/blogs/assets/91128330/10a31869-7b3e-411d-a4c8-b8e369967508)
The green ones are static addresses, it shows the offset relative to the process it's loaded into, in this case I have `client.dll+8E1540`, `client.dll+903B68` and `client.dll+9EEB38`. I'll add them to my address list by double clicking and keep them there.\
Now i'm going to try invalidate them, do whatever I can without closing the game to set that value to something else, preferably zero. The obvious would be to disconnect from the server.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/f5186918-c499-4340-bf21-7e8931f3ecf0)
I honestly wasn't expecting there to be any non-zero values but look where we are. Can remove that old faulty unreliable pointer.\
Now with two left, i'm going to load back into my trust dedicated server, watching my cheat engine values checking whether they change.\
And they're both valid!
![image](https://github.com/RegiSimkus/blogs/assets/91128330/df6516f6-6bfa-4d31-98ff-17303e633e0c)
\
Just for peace of mind to be extra sure, can take both of those addresses in my address table and tick the 'pointer' box and apply an offset of C8 to find our health again. As health is an integer, set that to 4 bytes, don't want hexidecimal. And wey hey! It's 100.\
\
Remember, it's not the 'Address' you want, it's the offset to the dll!
![image](https://github.com/RegiSimkus/blogs/assets/91128330/ec2b67d9-82f3-48cc-8a35-8ef8e141a500)
\
You can use this in cpp code using something like
```cpp
#include <Windows.h>
...
HMODULE hModule = GetModuleHandleA("client.dll");
if (!hModule) return; // validate ofc, shouldn't really be 0, but you might be loading your dll very early.
typedef Entity void*;
Entity** ppLocalPlayer = (Entity**)hModule + 0x903B68;
printf("LocalPlayer static address: %p\n", ppLocalPlayer);
Entity* pLocalPlayer = *ppLocalPlayer;
if (!pLocalPlayer)
  printf("Not valid.\n");
else
  printf("Player address and health: %p + 0x68 = %d\n", pLocalPlayer, *(int*)((uintptr_t)pLocalPlayer));```
