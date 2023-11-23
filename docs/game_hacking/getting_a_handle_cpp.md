# Introduction to creating external cheats in C++
Before we can start playing around with memory from within our external cheat, we first need to get a handle to the process we're trying to hack. 

## What are handles?
Without getting too technical, think of a handle as an object that identifies a program or a game we're trying to hack. We can then send that handle to other functions, such as ReadProcessMemory and WriteProcessMemory, which we will explore later.

## Source code of our target program

```cpp title="game.cpp"
#include <iostream>
#include <windows.h>
#include <TlHelp32.h>
#include <vector>
#include <chrono>
#include <thread>

struct Ammo {
    int buffer[4];
    int primaryWeaponAmmo = 100; // 0x10
};

struct Inventory {
    int buffer[3];
    Ammo* ammo; // 0xC
};

struct Player {
    int buffer[2];
    Inventory* inventory; // 0x8
};

struct Actor {
    int buffer;
    Player* player; // 0x4
};

Actor* localPlayer;

int main()
{
    SetConsoleTitleA("game.exe")
    
    localPlayer = new Actor();
    localPlayer->player = new Player();
    localPlayer->player->inventory = new Inventory();
    localPlayer->player->inventory->ammo = new Ammo();

    std::cout << "localPlayer pointer: " << "0x" << std::hex << &localPlayer << std::endl << std::endl;

    std::cout << "offsets are: " << "{ 0x4, 0x8, 0xC, 0x10 }" << std::endl << std::endl;

    std::cout << "primaryWeaponAmmo address: " << "0x" << std::hex << &localPlayer->player->inventory->ammo->primaryWeaponAmmo << std::endl << std::endl;

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(3000));

        std::cout << "primaryWeaponAmmo value: " << std::dec << localPlayer->player->inventory->ammo->primaryWeaponAmmo << std::endl;
    }
}
```

If you've read the [Multi-Level Pointers](multi_level_pointers.md) section, this code should be familiar. The only change is the `while` loop, which prints out the value of our `primaryWeaponAmmo` variable every 3 seconds (this will help us verify that our cheat is working when we start modifying memory), and the `SetConsoleTitleA("game.exe")` call which will set the title of the console application to "game.exe".

## Getting a handle to a process
Getting a handle to a process consists of a few steps. We can use the [OpenProcess()](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) function to get the handle, let's take a look at its definition:
```cpp
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```

After reading the official documentation, we can see that we have to use the values `PROCESS_VM_READ` and `PROCESS_VM_WRITE` for the `dwDesiredAccess` argument (because we want to read and write the memory). The second argument doesn't matter to us, so we can set it to false. However, we don't have the processId, so we need to find it using the [GetWindowThreadProcessId()](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getwindowthreadprocessid) function:

```cpp
DWORD GetWindowThreadProcessId(
  [in]            HWND    hWnd,
  [out, optional] LPDWORD lpdwProcessId
);
```

This function takes a pointer to a variable where it will store the process id and a handle to a window (HWND), which we can get with the [FindWindowA()](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-findwindowa) function:

```cpp
HWND FindWindowA(
  [in, optional] LPCSTR lpClassName,
  [in, optional] LPCSTR lpWindowName
);
```

The first argument doesn't matter to us, so we can set it to NULL. The second argument expects a window title, which in our source code we set to "game.exe", and that's what we are going to use.

To summarise: in order to get a handle to a process we need the process id of that process. In order to get the process id we need to get the handle to its window.

Knowing all this, we can start writing our cheat.

### Writing our cheat

```cpp title="external.cpp"
#include <windows.h>
#include <iostream>

int main(int argc, char** argv) {
	const char* windowName = "game.exe";

	HWND gameWindow = FindWindowA(NULL, windowName);

	DWORD gameProcessId = 0;
	GetWindowThreadProcessId(gameWindow, &gameProcessId);

	HANDLE hGame = OpenProcess(PROCESS_VM_READ | PROCESS_VM_WRITE, false, gameProcessId);

	if (!hGame) {
		std::cout << "Could not get the handle to " << windowName << ", make sure it's running!" << std::endl;

		return 1;
	}

	std::cout << "Successfully got the handle to " << windowName << ": " << "0x" << &hGame << std::endl;

	return 0;
}
```

First we find the window with the name "game.exe", then we use it to get the process id, then we use the process id to get the handle and we set the `PROCESS_VM_READ` and `PROCESS_VM_WRITE` access rights. After that we do a bit of error handling where we show an error message if the handle couldn't be obtained. 

Before you run this program, make sure you compile and run the source code of our target program shown at the very start of this post. If everything goes as planned, you should see this:

![handle](/assets/images/external-handle.png)

Great, we successfully got the handle to our game!

Now we can wrap this inside of a function to clean things up a bit:

```cpp title="external.cpp"
#include <windows.h>
#include <iostream>

HANDLE GetProcessHandle(DWORD dwDesiredAccess, const char* windowName) {
	HWND gameWindow = FindWindowA(NULL, windowName);

	DWORD gameProcessId = 0;
	GetWindowThreadProcessId(gameWindow, &gameProcessId);

	return OpenProcess(dwDesiredAccess, false, gameProcessId);
}

int main(int argc, char** argv) {
	const char* windowName = "game.exe";

	HANDLE hGame = GetProcessHandle(PROCESS_VM_READ | PROCESS_VM_WRITE, windowName);
	if (!hGame) {
		std::cout << "Could not get the handle to " << windowName << ", make sure it's running!" << std::endl;

		return 1;
	}

	std::cout << "Successfully got the handle to " << windowName << ": " << "0x" << &hGame << std::endl;

	return 0;
}
```

And now we are officially ready to start playing around with the memory!
