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
