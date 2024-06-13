---
title: A Data Structure You Haven't Heard Of
description: Showcasing the generational arena data structure and how it can be used for scenarios such as game development. It is also sometimes called a generational pool.
tags:
    - data structure
    - generational arena
    - game dev
    - C#
date: 2024-06-12
---

Imagine you are implementing an application, or in this case a game, that consists of many objects with dynamic lifetimes that also reference each other. To get rid of pointer related issues such as cache misses and invalid pointers you could store all objects in a big array and have them reference each other through indices, like this:

```csharp
public struct Missile(int damage)
{
    public int Damage { get; } = damage;

    //Points to an index in the EnemyContainer array
    public int TargetId { get; set; }
}

public struct Enemy
{
    public int Health { get; set; }
}

public struct EnemyContainer
{
    //Fixed size array
    public Enemy[] Enemies { get; set; }
}
```

This is sometimes good enough but this wouldn't be a blog post unless we could improve on it, right? In games you often want to avoid dynamic memory allocation so the arrays are usually fixed size, meaning we don't resize them and instead reuse indices. The problem with the above approach is that if an `Enemy` at `index 2` is destroyed we might spawn a new one at that same index. That would cause the `Missile` to follow a totally different enemy which might not be what we wanted, resulting in a bug.

![Example of an array of enemies](/images/array.png)

## Generational Arenas
We can solve the lifetime tracking issue and get constant time `O(1)` insertions, lookups and deletes by using a data structure called Generational Arena (sometimes referred to as Generation Pool). A Generational Arena is similar to a [Slot map](https://docs.rs/slotmap/latest/slotmap/) and uses the same concept as the data array but we introduce a generation counter for each index. Every time data is deleted at an index, we increase the generation counter at that index. When asking for data at an index, we check if the generation at the provided index is the same as the generation in the arena. If the data was deleted (and possibly reused), the generation would be higher, meaning the arena would return null.

If we wanted to get the data at index 2 but provide a generation of 3 we would get null because the data has been removed since we got our handle. However, if we provide a generation of 4 we would get the data because the data has not been removed meaning our view of that data is still up to date.

![Example of a generational arena of enemies](/images/generational_arena.png)

To make insertion an O(1) operation, a [freelist](https://en.wikipedia.org/wiki/Free_list) is used. It's basically just a stack that we push to when data is removed and pop from when data is added. Sometimes, you'd want dynamic resizing of your arena but that's not something I need for my use case. Implementing that and optimising the freelist is left as an exercise for the interested reader.

Here's the implementation and as promised, it's less than 50 lines of code. The number of lines would probably increase a bit if you do the suggested exercise above but not by much.

```csharp
//Note: Safety checks are removed for brevity
public class GenerationalArena<T>(uint maxCapacity)
{
    private readonly T[] _data = new T[maxCapacity];
    private readonly uint[] _generations = new uint[maxCapacity];

    private uint _nrOfUsedIndices = 0;

    //Not fixed size in this example, left as an optimizing exercise for the reader
    private readonly Stack<uint> _freeList = new();

    public Handle<T> Add(T data)
    {
        var freeIndex = _freeList.TryPop(out uint recycledIndex) 
            ? recycledIndex 
            : _nrOfUsedIndices;

        _data[freeIndex] = data;
        _nrOfUsedIndices++;

        return new Handle<T>(freeIndex, _generations[freeIndex]);
    }

    public T Get(Handle<T> handle)
    {
        return handle.Generation == _generations[handle.Index] 
            ? _data[handle.Index] 
            : default;
    }

    public void Remove(Handle<T> handle)
    {
        _generations[handle.Index]++;
        _freeList.Push(handle.Index);
    }
}

//T for type safety so you can't accidentally use a handle with the wrong arena
public readonly struct Handle<T>(uint index, uint generation)
{
    public uint Index { get; } = index;
    public uint Generation { get; } = generation;
}
```

Here's a silly example of how it could be used:
```csharp
//Silly example size to showcase the generation count
var generationalArena = new GenerationalArena<Enemy>(1);

var missileFiredByPlayer = new PlayerMissile(1);
missileFiredByPlayer.Target = RunGoalSeekingMissileLogic();

//Destroys the enemy and removes it from the arena. This increases the generation counter
Explode(missileFiredByPlayer);

//Enemy will be null because the generation counter is wrong
var enemy = generationalArena.Get(missileFiredByPlayer.Target);

//newEnemyHandle will have the same Index as the previous target but a different generation
var newEnemyHandle = generationalArena.Add(new Enemy()); 
```

As you can see it's a really useful data structure with constant time operations that does not take many lines of code to implement.

Enjoy!