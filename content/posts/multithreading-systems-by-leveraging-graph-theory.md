---
title: Multithreading Systems by Leveraging Graph Theory
description: In this blog post we leverage graph theory by performing a topological sort with Kahn's algorithm to be able to multithread a list of systems in our game.
tags:
    - multithreading
    - game
    - game dev
    - systems
    - topological sort
    - kahn's algorithm
    - graph theory
    - C#
date: 2024-05-31
---

A common ambition in game development is to multithread parts of your game to take advantage of today's multi-core processors. The challenge is that in games, you can have hundreds of different systems that write to and read from data. How do we multithread these hundreds of systems while avoiding race conditions that occur when multiple threads write to and read from the same data? In the example below, we have 4 different systems which don't seem too hard to multithread, but in real scenarios it will become unmanageable faster than you think.

```yaml
PoisonSystem: 
  Read:
    - PoisonCounter
  Write:
    - Health

GameOverSystem:
  Read:
    - Health
  Write:
    - GameState

HealthBarSystem:
  Read:
    - Health
  Write:
    - GUI

MovementSystem:
  Read:
    - Input
  Write:
    - Position
```

To add even more complexity, there are usually rules for the order in which systems should be able to run. We use the following:
- Systems that write to data should run before systems that read from that same data. In our example, `PoisonSystem` should run before `GameOverSystem` and `HealthBarSystem` because it writes to `Health`, while the others read from `Health`.
- Systems that read from the same data can run in parallel. In our example `GameOverSystem` and `HealthBarSystem` can run in parallel and the same goes for `MovementSystem` because it doesn't read any data the other systems write to.
- We want to forbid two systems from writing to the same data because that would mean both would need to run before the other to avoid a race condition and that's impossible.


## Graph theory to the rescue
> A graph is a structure amounting to a set of objects in which some pairs of the objects are in some sense "related". The objects are represented by abstractions called vertices and each of the related pairs of vertices is called an edge.[^1]

This sounds similar to our problem, doesn't it? If we start thinking of the problem as a graph where our systems are the vertices and their data dependencies as edges it becomes easier to reason about. Below is what it would look like if we placed our systems in a graph.

![Directed graph of the 4 example systems and their data dependencies](/images/graph.svg)

To achieve this graph in code we just need to loop over every pair of systems and compare their read and writes. Which results in the following code:

```csharp
AssignEdgesToVertices([
    new("PoisonSystem", "PoisonCounter", "Health"),
    new("GameOverSystem", "Health", "GameState"),
    new("HealthBarSystem", "Health", "GUI"),
    new("MovementSystem", "Input", "Position")]);

/* 
 * Goes over all vertices and assigns edges between them depending on
 * if they access the same data or not
 */
static void AssignEdgesToVertices(GameSystem[] vertices)
{
    for (int i = 0; i < vertices.Length; i++)
    {
        for (int j = i + 1; j < vertices.Length; j++)
        {
            var system = vertices[i];
            var nextSystem = vertices[j];

            /*
             * If System writes to the data NextSystem writes:
             * [System]
             *   | ^
             *   v |
             * [NextSystem]
             */
            if (system.Write == nextSystem.Write)
            {
                system.AddOutgoingEdgeTo(nextSystem);
                nextSystem.AddOutgoingEdgeTo(system);
            }

            /*
             * If System writes to the data NextTask reads:
             * [System]
             *   |
             *   v
             * [NextSystem]
             */
            if (system.Write == nextSystem.Read)
            {
                system.AddOutgoingEdgeTo(nextSystem);
            }

            /*
             * If System reads from the data that NextSystem writes:
             *  [System]
             *    ^
             *    |
             *  [NextSystem]
             */
            if (system.Read == nextSystem.Write)
            {
                nextSystem.AddOutgoingEdgeTo(system);
            }
        }
    }
}

//This is a simplified example for brevity, NOT real production code :D
public class GameSystem(string name, string read, string write)
{
    public string Name { get; } = name;

    public string Read { get; } = read;
    public string Write { get; } = write;

    public List<GameSystem> IncomingEdges { get; } = [];
    public List<GameSystem> OutgoingEdges { get; } = [];

    //These will come in handy later on, trust me
    public int NrOfIncomingEdges { get; set; }
    public JobHandle JobHandle { get; set; }

    //Helper method to make the graph creation code easier to read
    public void AddOutgoingEdgeTo(GameSystem toVertex)
    {
        OutgoingEdges.Add(toVertex);
        toVertex.IncomingEdges.Add(this);
        toVertex.NrOfIncomingEdges++;
    }

    //Simulate work being done
    public void DoWork()
    {
        Console.WriteLine($"{Thread.CurrentThread.Name}: {Name}");
    }
}
```

## Topological sorting
The careful reader should have noticed that the code does not follow one of our previously mentioned rules:
> We want to forbid two systems from writing to the same data because that would mean both would need to run before the other to avoid a race condition and that's impossible.

This is not great because this is a rule we **have** to follow to not get race conditions that would **crash** our entire application. For example if we added a new system called `BulletSystem` that also writes to `Health` it would result in a cycle where `PoisonSystem` and `BulletSystem` need to run before the other:
![Directed graph of the 4 example systems and their data dependencies](/images/graph-loop.svg)


This is where we can leverage a technique called topological sorting:

> A topological sort or topological ordering of a directed graph is a linear ordering of its vertices such that for every directed edge (u,v) from vertex u to vertex v, u comes before v in the ordering.[^2]

That solves our issue of not knowing in which order we should run our systems. Additionally, it solves the problem of potential cycles:

>  A graph that has a topological ordering cannot have any cycles, because the edge into the earliest vertex of a cycle would have to be oriented the wrong way. Therefore, every graph with a topological ordering is acyclic. Conversely, every directed acyclic graph has at least one topological ordering.[^3]

Knowing that performing a topological sort will solve all of our problems, we will jump straight away into the implementation! We will implement Kahn's algorithm[^4], which is a simple breadth-first search algorithm. It works by finding all vertices with no incoming edges, adding them to the sorted list, removing them from the graph and then updating the incoming edges of all its connected vertices. This is then repeated until all vertices have been ordered.

```csharp
//Topological with Kahn's algorithm
static List<GameSystem> TopologicalSort(GameSystem[] graph)
{
    if (graph.Length == 0)
    {
        return [];
    }

    //All vertices with no incoming edges are sources that should be sorted first
    var queue = new Queue<GameSystem>();
    foreach (var node in graph.Where(n => n.NrOfIncomingEdges == 0))
    {
        queue.Enqueue(node);
    }

    var sorted = new List<GameSystem>();
    while (queue.Count > 0)
    {
        var source = queue.Dequeue();
        sorted.Add(source);

        //We added the vertice to the sorted list meaning we need to remove it from the graph.
        //We do that by reducing the nr of incoming edges for all the outgoing edges.
        foreach (var outgoingNode in source.OutgoingEdges)
        {
            outgoingNode.NrOfIncomingEdges--;

            //If the outgoing edge has no incoming edges it's added to the sorted list
            if (outgoingNode.NrOfIncomingEdges == 0)
            {
                queue.Enqueue(outgoingNode);
            }
        }
    }

   /* 
    * Topological sort is possible when the graph is a
    * Directed Acyclic Graph (meaning no cycles).
    * If there is a cycle it means vertices have dependencies
    * on each other making it impossible to sort the vertices linearly.
    */
    if (sorted.Count != graph.Length)
    {
        throw new Exception("Not a Directed Acyclic Graph (DAG) because it contains cycles!");
    }

    return sorted;
}
```

If we perform the topological sort on our previously mentioned list of systems we get the following order:
```plaintext
PoisonSystem -> MovementSystem -> GameOverSystem -> HealthBarSystem
```

## Multithreading the systems
With our list of systems sorted we now have everything we need to multithread the execution of our systems. The choice of library is up to you (you can even use `System.Threading.Tasks.Task` if you'd like) but I'll be using a custom implementation with a few worker threads. The code will differ depending on how you implement the multithreading but in my case it looks like this:

```csharp
var topologicalSortedSystems = TopologicalSort(systems);
foreach (var system in topologicalSortedSystems)
{
    if (system.IncomingEdges.Count == 0)
    {
        //We don't have any incoming edges, just schedule
        system.JobReference = JobScheduler.Schedule(system);
    }
    else
    {
        //The list is sorted, so we are guaranteed that all incoming edges already have a JobReference
        var jobReferencesOfIncomingEdges = system.IncomingEdges.Select(ie => ie.JobReference).ToArray();

        //We combine all references to one to make it easier to handle
        var combineDependencies = JobScheduler.Combine(jobReferencesOfIncomingEdges);

        //Our JobReference is our own work that is scheduled after all of or incoming edges work
        system.JobReference = JobScheduler.Schedule(system, combineDependencies);
    }
}

JobScheduler.StartQueuedWork();
JobScheduler.WaitForAll(sortedSystems.Select(n => n.JobReference).ToList());
```

The code above outputs:
```plaintext
Thread 1: PoisonSystem
Thread 3: MovementSystem
Thread 1: GameOverSystem
Thread 2: HealthBarSystem
```

Exactly which thread is being used for which system isn't important in this case, but the order is. `MovementSystem` and `PoisonSystem` are run in parallel while `GameOverSystem` and `HealthBarSystem` have to wait for `PoisonSystem` to complete. After it's complete they can both run in parallel on different threads. This means we have accomplished everything we set out to do! The implementation details of the custom job system might become a subject for a future blog post, but that will have to wait.

Thanks for reading!

[^1]: https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)
[^2]: https://en.wikipedia.org/wiki/Topological_sorting
[^3]: https://en.wikipedia.org/wiki/Directed_acyclic_graph
[^4]: https://dl.acm.org/doi/pdf/10.1145/368996.369025