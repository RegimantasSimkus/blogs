## Introduction

The [Steam Overlay](https://partner.steamgames.com/doc/features/overlay) is an interface for interacting with Steam from within a game. The overlay works by hooking onto the creation of the rendering device to get a handle to the created device in order to hook the rendering methods found within the VTable.

The Steam Overlay is loaded into the game as GameOverlayRenderer.dll or GameOverlayRenderer64.dll depending on whether the game is an x86 or x64 process.

Since what i'm looking to do is hook the overlay for CS:GO which is an x86 process I will be reverse engineering GameOverlayRenderer.dll to get a deeper understanding of how it works.

## Reverse Engineering with IDA Pro
Having opened the dll with IDA Pro you find that there are 14 exported functions as well as the entrypoint for the binary. 
![image](https://github.com/RegiSimkus/blogs/assets/91128330/9477cb3b-c831-4864-a8e7-7fad4bf7ce19)

None of these look too interesting except for OverlayHookD3D which doesn't look entirely promising having looked further into it.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/7c967042-e224-4afb-a540-cfc582772664)

Since I know that CS:GO uses DirectX9 I can look for references to the string "d3d9.dll" which can be cross referenced to help find the code responsible for hooking Direct3DCreate9.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/6ee7af60-c110-4fab-af69-c1d58780d255)
![image](https://github.com/RegiSimkus/blogs/assets/91128330/bded2a1f-f30e-47c1-b459-a51b0263a108)

The string cross reference took me to this\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/21552949-6ba8-489f-9b08-f39f08ec11e7)\
You can see that `v15` is defined as the result of a function call in which the argument is "d3d9.dll" which is safe to assume that it is a wrapper for GetModuleHandle, I can be sure of this by looking in the code of the function to determine whether i'm right.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/7585f0b8-0fe1-4c08-9bb4-953644dda246)\
Which looks like i'm right, so I can rename the symbol for this function to be able to identify it again in the future by name.
The same is right for `v16`'s function which is just another wrapper for `GetProcAddress`.

The code as I interpret it now is that it does something along the lines of
```
HMODULE hD3D9 = GetModuleHandle("d3d9.dll");
if (hD3D9)
{
  void* pDirect3DCreate9 = GetProcAddress(hD3D9, "Direct3DCreate9");
  if (pDirect3DCreate9 && !sub_6F540(pDirect3DCreate9, (int)sub_69Fc0))
  {
    DebugPrint("Preparing to hook...\n");
    HookFunction(GetModuleHandleA, pDirect3DCreate9, dword_1076b8, 0);
  }
}
```
Following cross references to `dword_1076b8` suggests that it is where the original function is stored after being hooked.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/28e86a79-d44e-4bba-a292-d9e9586d16fa)

Now that I am certain of the function responsible for hooking, I can see where else the function is being used.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/d6ca0ba7-312f-46f2-8cc9-6f9a8afb4ef6)\
Which provides an extensive list of results which doesn't give too much information about the hooking of the IDirect3DDevice9 virtual methods.

## Reverse Engineering with C++ and Cheat Engine
Since the hooking of a function is the process of inserting a JMP at the start which goes to your own hook we can see this in memory and follow the jump which will lead us straight to Steam's rendering hook.
In order to find the function I'm looking for, in this case, IDirect3DDevice9::Present(), I need to find a pointer to the game's D3DDevice. There is a way of dynamically fetching a pointer to this by creating your own D3D device which can be used to create the interface that points to the same handle as CS:GO's.

The same method of retrieving the device is used in my open source CS:GO cheat which can be found [here](https://github.com/BullyHunter32/Bullyware-CSGO/blob/3d51da8cbaacb9fb436a00252545d03a8af8c4a0/Bullyware-CSGO/Bullyware/Render/D3DDevice.cpp#L6-L33).
```
IDirect3DDevice9* GetD3DDevice()
{
	if (g_pDevice) return g_pDevice;

	using namespace Bullyware;
	if (!hWnd)
		return nullptr; // cant retrive device without hWnd

	IDirect3D9* pD3D = Direct3DCreate9(D3D_SDK_VERSION);
	D3DPRESENT_PARAMETERS params = {};
	ZeroMemory(&params, sizeof(D3DPRESENT_PARAMETERS));
	params.Windowed = TRUE;
	params.SwapEffect = D3DSWAPEFFECT_DISCARD;
	params.AutoDepthStencilFormat = D3DFMT_UNKNOWN;
	params.hDeviceWindow = hWnd;

	if (HRESULT res = pD3D->CreateDevice(D3DADAPTER_DEFAULT, D3DDEVTYPE_HAL, hWnd, D3DCREATE_SOFTWARE_VERTEXPROCESSING, &params, &g_pDevice) < 0)
	{
		params.Windowed = !params.Windowed;
		pD3D->CreateDevice(D3DADAPTER_DEFAULT, D3DDEVTYPE_HAL, hWnd, D3DCREATE_SOFTWARE_VERTEXPROCESSING, &params, &g_pDevice);
	}

	// clean up some memory as we dont need d3d anymore
	if (pD3D)
		pD3D->Release();

	return g_pDevice;
}
```
![image](https://github.com/RegiSimkus/blogs/assets/91128330/2befe452-b652-4dff-901b-f30967492a60)\
After retrieving a handle to the device we can then look through the assembly of the virtual methods as listed in the VTable. To find out which function I need to look at I can simply just reference the definition of the class as defined in `d3d9.h`.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/c104d204-7def-441e-a554-43f72e4a7ff4)\
The function is in index 17 meaning the function can be found at `VTable+(17*4)`.

Using the Sturcture disect view of Cheat Engine & inputting the address of the device I can look for the 17th function which is at offset 17\*4/68/0x44.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/59244937-c9dd-4e4e-84f0-232ba5094c6f)\
Using the Memory View which gives a view of the live code in RAM we can see a JMP to an address with no symbols loaded which is likely an area in memory that was dynamically allocated by the OS.
![image](https://github.com/RegiSimkus/blogs/assets/91128330/78607486-1ea4-47bf-8543-5f76b6f4e2f9)\
Following the JMP takes us to another JMP to the gameoverlayrenderer binary.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/962a8e4d-9bc6-4c03-b806-7fb7976e539d)\
which indicates that this is the hook.\
![image](https://github.com/RegiSimkus/blogs/assets/91128330/0c2dc160-2e83-4a89-9396-be15d3446ede)

This means that we can dynamically and easily hook the Steam Overlay's Present by either generating a code signature, or more reliably for CSGO, can follow the two jumps.

## Conclusion

I have defined a wrapper for following the two JMPs as
```
	inline PVOID EvaluateRelASM(PVOID pAddy)
	{
		uintptr_t addy = (uintptr_t)pAddy + 1; // E9 <relative offset>
		addy = addy + *(int*)addy + 4;
		return (PVOID)addy;
	}

	template <typename T>
	T EvaluateRelASM(PVOID pAddy)
	{
		return (T)EvaluateRelASM(pAddy);
	}
```

Which can be used like so\
```
	// IDirect3DDevice9::Present
	PVOID pPresent = (*(void***)g_pDevice)[17];

	// Following the two jumps that the Steam present hook implement
	uintptr_t pResolvedSteamPresent = Utils::EvaluateRelASM<uintptr_t>(Utils::EvaluateRelASM(pPresent));

	// Hooking steam's present hook
	Present = new CTrampHook((PVOID)pResolvedSteamPresent, (PVOID)hkPresent, 6, (PVOID*)(&oPresent));
```
And just like that, we have the address of the hook, the IDirect3DDevice9 interface, and the static address of the function relative to gameoverlayrenderer.dll which is `0x66af0`.
