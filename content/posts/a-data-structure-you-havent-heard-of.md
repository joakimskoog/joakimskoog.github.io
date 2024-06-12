---
title: A Data Structure You Haven't Heard Of
description: Showcasing the generational arena data structure and how it can be used for scenarios such as game development. It is also sometimes called a generational pool.
tags:
    - data structure
    - generational arena
    - generational pool
    - game dev
    - C#
date: 2024-06-12
---

If I asked you to implement a goal-seeking missile, how would you do that? If you're a software engineer, it could look something like this:

```csharp
public class Missile(int damage, Enemy target)
{
    public int Damage { get; set; } = damage;
    public Enemy Target { get; set; } = target;
}

public class Enemy(int health)
{
    public int Health { get; set; } = health;
}
```

If you're a game developer, you might instead implement it similar to this:
```csharp
public struct Missile(int damage)
{
    public int Damage { get; } = damage;
    public Handle<Enemy> Target { get; set; }
}

public struct Enemy
{
    public int Health { get; set; }
}

public struct Handle<Enemy>(int index)
{
    //Corresponds to an array index in a pool
    public int Index { get; set; } = index;
}
```

Why the difference? The first approach is fine if you're developing a regular business application but games have different requirements:
- An array index is safe to pass to other threads
- [Pooling](https://en.wikipedia.org/wiki/Pool_(computer_science)) is heavily used to reduce dynamic memory allocations that could degrade the game's performance.
- Following pointers all over memory results in cache misses and worse performance

![Example of a pool of enemies](/images/pool.png)

The pool solves all of our performance issues, but by using a pool we have introduced a new **problem**. Can you spot it? What happens if the `Enemy` dies and the handle is reused? Our `Missile` would still have the same `Handle<Enemy>` but `Index` would point to a different `Enemy` causing all sorts of bugs.

## Generational pools
We can solve the lifetime tracking problem mentioned above by using a neat data structure called Generational Pool (also sometimes referred to as Generational Arena) that we can implement in less than 50 lines of C#. A Generational Pool is identical to a regular pool except that we introduce a generation counter for each index. Every time data is deleted at an index, we increase the generation counter at that index. When asking for data at an index, we check if the generation at the provided index is the same as the generation in the pool. If the data was deleted (and possibly reused), the generation would be higher, meaning the pool would return null.

If we wanted to get the data at index 2 but provide a generation of 3 we would get null because the data has been removed since we got our handle. However if we provide a generation of 4 we would get the data because the data has not been removed meaning our view of that data is still up to date.

![Example of a generational pool of enemies](/images/generational_pool.png)

To make insertion an O(1) operation, a [freelist](https://en.wikipedia.org/wiki/Free_list) is used. It's basically just a stack that we push to when data is removed and pop from when data is added. Usually, you'd want dynamic resizing of your pool but that's not something I need for my use case. Implementing that, optimising the freelist and prefilling the pool with data is left as an exercise for the interested reader.

Here's the implementation and as promised it's less than 50 lines of code. The number of lines would probably increase a bit if you do the suggested exercise above but not by much.

```csharp
public class GenerationalPool<T>(uint maxCapacity)
{
    private readonly T[] _pool = new T[maxCapacity];
    private readonly uint[] _generations = new uint[maxCapacity];

    private uint _nrOfUsedIndices = 0;
    private readonly Stack<uint> _freeList = new();

    //Dynamic resizing removed for brevity
    public Handle<T> Add(T data)
    {
        var freeIndex = _freeList.TryPop(out uint recycledIndex) 
            ? recycledIndex 
            : _nrOfUsedIndices;

        _pool[freeIndex] = data;
        _nrOfUsedIndices++;

        return new Handle<T>(freeIndex, _generations[freeIndex]);
    }

    public T Get(Handle<T> handle)
    {
        if (handle.Generation != _generations[handle.Index])
        {
            return default;
        }

        return _pool[handle.Index];
    }

    public void Remove(Handle<T> handle)
    {
        _generations[handle.Index]++;
        _freeList.Push(handle.Index);
    }
}

//T for type safety so you cant accidentally use a handle with the wrong pool
public readonly struct Handle<T>(uint index, uint generation)
{
    public uint Index { get; } = index;
    public uint Generation { get; } = generation;
}
```

Silly example on how it could be used:
```csharp
//Silly example size to showcase the generation count
var generationalPool = new GenerationalPool<Enemy>(1); 

var missileFiredByPlayer = new PlayerMissile(1);
missileFiredByPlayer.Target = RunGoalSeekingMissileLogic();

//Destroys the enemy and removes it from the pool. This increases the generation counter
missileFiredByPlayer.Explode(); 

//Enemy will be null because the generation counter is wrong
var enemy = generationalPool.Get(missileFiredByPlayer.Target);

//newEnemyHandle will have the same Index as the previous target but a different generation
var newEnemyHandle = generationalPool.Add(new Enemy()); 
```

As you can see it's a really useful data structure that does not take many lines of code at all to implement.

Enjoy!