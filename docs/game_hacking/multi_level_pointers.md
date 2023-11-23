# Introduction to multi-level pointers (work in progress)

## What are multi-level pointers?
Multi-level pointers (MLPs) are actually really simple, as tricky and complicated as they might seem when you first start learning about them (don't worry if you're struggling, I did too). MLPs are just pointers pointing to other pointers, it's essentially a chain of pointers. It should make more sense after taking a look at the example below.

## Example
Consider the following code:

```cpp title="mlp.cpp"
// ...

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

// ...
```

Here we've defined 4 structs. Each has a buffer property for simulating other properties inside a struct, and a pointer to another struct. The comments represent offsets.

In the `Actor` struct, the offset for the `player` pointer is 0x4, because the `buffer` property above it is an int, as ints are 4 bytes long the offset is 0x4. But what about the `Inventory` struct, why is the offset for the `ammo` pointer 0xC, shouldn't it be 0x12? And yes, you would be right, it should be 12, and it actually is! As you've probably guessed, pointers are represented as hexadecimal values, and C in hexadecimal equates to 12 in decimal. Same logic applies to the `Ammo` struct as well. The offset may look like 10 in decimal, but as that value is in hexadecimal, the decimal value is 16.

Now let's take a look at how we're going to initialise all these structs:

```cpp title="mlp.cpp"
// ...

Actor* localPlayer;

int main()
{
    localPlayer = new Actor();
    localPlayer->player = new Player();
    localPlayer->player->inventory = new Inventory();
    localPlayer->player->inventory->ammo = new Ammo();

//..
```

As you can see, the code isn't anything special. All we did is define a global localPlayer variable that's a pointer to the `Actor` struct, then we instantiated all of the structs.

Now let's add some code that will help us with debugging:

```cpp title="mlp.cpp"
// ...

    std::cout << "localPlayer pointer: " << "0x" << std::hex << &localPlayer << std::endl << std::endl;

    std::cout << "offsets are: " << "{ 0x4, 0x8, 0xC, 0x10 }" << std::endl << std::endl;

    std::cout << "primaryWeaponAmmo address: " << "0x" << std::hex << &localPlayer->player->inventory->ammo->primaryWeaponAmmo << std::endl << std::endl;

    getchar();
}
```

All we're doing here is printing out the localPlayer pointer, the offsets and the address of the primaryWeaponAmmo field. The call to `getchar()` is there to prevent the program from ending straight away.

Here's what the entire main function looks like:

```cpp title="mlp.cpp"
Actor* localPlayer;

int main()
{
    localPlayer = new Actor();
    localPlayer->player = new Player();
    localPlayer->player->inventory = new Inventory();
    localPlayer->player->inventory->ammo = new Ammo();

    std::cout << "localPlayer pointer: " << "0x" << std::hex << &localPlayer << std::endl << std::endl;

    std::cout << "offsets are: " << "{ 0x4, 0x8, 0xC, 0x10 }" << std::endl << std::endl;

    std::cout << "primaryWeaponAmmo address: " << "0x" << std::hex << &localPlayer->player->inventory->ammo->primaryWeaponAmmo << std::endl << std::endl;

    getchar();
}
```

## Testing with Cheat Engine
This is what the output should be after we run our program (your addresses will probably differ):

![mlp console](/assets/images/mlp-console1.png)

Perfect, everything seems to be running, let's have some fun with Cheat Engine now. Let's add the primaryWeaponAmmo address into CE:

![mlp cheat engine](/assets/images/mlp-ce1.png)

Great! After typing in the address we see that Cheat Engine resolves it to the value 100, which is exactly what we set in our `Ammo` struct. Now let's restart our program and see what's going to happen:

![mlp console](/assets/images/mlp-console2.png)

Hmm, okay, our localPlayer pointer is the same, but our primaryWeaponAmmo address has changed. Let's re-attach Cheat Engine to our program and see what's going to happen:

![mlp cheat engine](/assets/images/mlp-ce2.png)

Uh oh! That's not good. As the address of the primaryWeaponAmmo variable has changed, the address we used previously doesn't work anymore. This is due to Dynamic Memory Allocation (DMA).

## Dynamic Memory Allocation (DMA)
Take a look at this code:

```cpp title="mlp.cpp"
localPlayer = new Actor();
localPlayer->player = new Player();
localPlayer->player->inventory = new Inventory();
localPlayer->player->inventory->ammo = new Ammo();
```

Every time we used the `new` keyword to initialise an instance of a struct, our program asked the OS to allocate some memory. That memory gets allocated on the [heap](https://www.programmerinterview.com/data-structures/difference-between-stack-and-heap), and the issue is, our program can't pick where to allocate the memory, it can only ask the OS to allocate it. That is an issue for us, because we can't guarantee that the OS will allocate the memory on the same addresses every time we initialise a struct.

Now, you might be asking yourself: Okay, but why is our `localPlayer` address the same despite all this DMA stuff? 

And that's a great question, which we have the answer to: We defined `localPlayer` as a global variable, and global variables are stored in something called the data segment. The addresses of global variables are determined at **compile time**, which means that every time we run the program the address will stay the same, despite DMA, because the address has already been determined when we compiled the program. However, if you were to make a change and re-compile the program, the address of `localPlayer` will change. This is why you need to find new offsets when a game updates (sometimes, but more often than not).

## Using multi-level pointers
When we ran our program for the second time, we noted that the `localPlayer` pointer has stayed the same. Which is great news for us, because we can use that fact to find our primaryWeaponAmmo value. Let's see how that's done in Cheat Engine:

![mlp cheat engine](/assets/images/mlp-ce3.png)

We put in the localPlayer address in the bottom field, then we add all four of our offsets. Cheat Engine gives us a breakdown of what it's doing.

!!! note
    Since we wrote this program ourselves, we know all of the offsets. In a real life scenario, you'd need to find them out manually. Check out [Defeating DMA](/binary_exploatation/buffer_overflow) to learn how to do that.

Our 0x9A5434 address (localPlayer) points to 0x00EFB6B0, then when we add the 0x4 offset to that address [0x00EFB6B0 + 4], which then points to 0x00EFB518. Then we add the 0x8 offset to that address [0x00EFB518 + 8], which then points to 0x00EF4738, so on and so forth, until we hit our final address (0x00EF4F38) which holds our primaryWeaponAmmo value.

Now, if we were to restart our program, all of these addresses (0x00EFB6B0, 0x00EFB518, 0x00EF4738, etc...) would be different, which is exactly the reason why we need a static base address (the localPlayer address in our case), and all of the offsets. We can then use that base address and the offsets to create a path to our desired value. 

As we do have a static base address and the offsets, we can safely assume that Cheat Engine will successfully get our primaryWeaponAmmo value even after we restart our program. Restart your program and re-attach Cheat Engine:

![mlp cheat engine](/assets/images/mlp-ce4.png)

And sure enough, it works! Let's take a look at how CE is resolving the pointers:

![mlp cheat engine](/assets/images/mlp-ce5.png)

As you can see, the pointers are indeed different, but despite that CE still managed to get our primaryWeaponAmmo value correctly. It was able to do that because it created a path from our static localPlayer address to the primaryWeaponAmmo variable using the offsets.

## The End
And there you go! Now you (hopefully) understand multi-level pointers. If you don't, feel free to re-read this post multiple times and use the resources I linked below.

## Resources
- [Multi Level Pointers In Depth Analysis Tutorial](https://www.youtube.com/watch?v=0iOxUOaogb8)
- [Dynamic Memory Allocation by Game Hacking Academy](https://gamehacking.academy/lesson/2/8)
- [What's the difference between a stack and a heap?](https://www.programmerinterview.com/data-structures/difference-between-stack-and-heap)
