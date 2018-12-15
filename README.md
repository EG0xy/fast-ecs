# fast-ecs
A C++17 fast, storage-wise, header-only Entity Component System library.

* fast-ecs is **fast** because it is cache-friendly: it organizes all the information it needs to iterate the entities and read the components in one single array. In my computer, it takes 0.005 us to iterate each entity (this is 0.000005ms!).
* fast-ecs is **storage-wise** because it organizes all components in one single array without gaps. This way, no space is wasted.
* fast-ecs is **header-only** - just include the header below and you're good to go! No need to link to external libraries.

This library was tested with the `g++` and `clang++` compilers.

### [Download the header file here!](https://raw.githubusercontent.com/andrenho/fast-ecs/master/fastecs.hh)

# Example

```C++
#include <iostream>

#include "fastecs.hh"

//
// COMPONENTS
//

struct Position {
    Position(float x, float y) : x(x), y(y) {}
    float x, y;
};

struct Direction {
    Direction(float angle) : angle(angle) {}
    float angle;
};

// 
// ENGINE
//

using MyEngine = ECS::Engine<class System, ECS::NoQueue, Position, Direction>;

//
// SYSTEMS
//

class System { 
public:
    virtual void Execute(MyEngine& e) = 0;
    virtual ~System() {}
};

class PositionSystem : public System {
public:
    void Execute(MyEngine& e) override {
        e.ForEach<Position>([](size_t entity, Position& pos) {
            pos.x += 1;
            std::cout << "Entity " << entity << " position.x was " << pos.x -1 <<
                         " but now is " << pos.x << ".\n";
        });
    }
};

class DirectionSystem : public System {
public:
    void Execute(MyEngine& e) override {
        e.ForEach<Direction>([](size_t entity, Direction& dir) {
            std::cout << "Entity " << entity << " direction is " << dir.angle << ".\n";
        });
    }
};

//
// MAIN PROCEDURE
//

int main()
{
    MyEngine e;

    size_t e1 = e.AddEntity(),
           e2 = e.AddEntity();

    e.AddComponent<Position>(e1, 20.f, 30.f);
    e.AddComponent<Direction>(e1, 1.2f);

    e.AddComponent<Position>(e2, 40.f, 50.f);
    e.GetComponent<Position>(e2).x = 100.f;

    e.AddSystem<PositionSystem>();
    e.AddSystem<DirectionSystem>();

    for(auto& sys: e.Systems()) {
        sys->Execute(e);
    }
}
```

The result is:

```
Entity 0 position.x was 20 but now is 21.
Entity 1 position.x was 100 but now is 101.
Entity 0 direction is 1.2.
```

# API

Engine management:

```C++
Engine<System, EventType, Components...>();   // create a new Engine
// `System` is the parent class for all systems. 
//     Use `ECS::NoSystem` if you don't want to use any systems.
// `EventType` is a variant type that contains all event types.
//     Use `ECS::NoQueue` if you don't want to use an event queue.
// `Components...` is a list of all components (structs) 
//     that can be added to the Engine.

// Since you'll want to use the engine declaration everywhere
// (pass to functions, etc), it is better to use a type-alias:
using MyEngine = ECS::Engine<System, EventType, Position, Direction>;
```

Entity management:

```C++
size_t AddEntity();                // create a new entity, and return that entity identifier number
void   RemoveEntity(Entity ent);   // delete an entity
```

Component management:

```C++
// A component is simply a struct.

struct Position {
    double x, y;
    Position(double x, double y) : x(x), y(y) {}
};

// More complex components can be used. The destructor will be 
// called when the component is destroyed.

struct Polygon {
    vector<Point> points = {};
};
```

Avoid using pointers in components, as it defeats the porpouse of the high speed array of this library.

Also, remember that entities and components might be moved within the array, so pointers to the components won't work. Always refer to the entities by their number.

```C++
C&   AddComponent<C>(size_t entity, ...);   // add a new component to an entity, calling its constructor
void RemoveComponent<C>(size_t entity);     // remove component from entity

bool HasComponent<C>(size_t entity);        // return true if entity contains a component
C&   GetComponent<C>(size_t entity);        // return a reference to a component

// When getting a component, it can be edited directly:
e.GetComponent<Position>(my_entity).x = 10;

// `GetComponentPtr` will return a pointer for the component, or nullptr if it doesn't exist.
// Thus, it can be used as a faster combination of `HasComponent` and `GetComponent`
C*   GetComponentPtr<C>(size_t entity);
```

Iterating over entities: 

```C++
void ForEach<C...>([](size_t entity, ...);

// Example:
e.ForEach<Position, Direction>([](size_t ent, Position& pos, Direction& dir) {
    // do something
});

// There's also a const version of ForEach: 
e.ForEach<Position, Direction>([](size_t ent, Position const& pos, Direction const& dir) {
    // do something
});
```

Management:

After deleting and readding many entities and components, the component array will become fragmented. So, eventually, is a good idea to compress the array, removing the spaces that were left:

```C++
void Compress();
```

Systems:

```C++
// all systems must inherit from a single class
struct System {
    virtual void Execute(MyEngine& e) = 0;
    virtual ~System() {}
}

struct MySystem : public System {
    void Execute(MyEngine& e) override {
        // do something
    }
};

System&         AddSystem<S>(...);    // add a new system (call constructor)
System&         GetSystem<S>();       // return a reference to a system
vector<System*> Systems();            // return a vector of systems to iterate, example:

for(auto& sys: e.Systems()) {
    sys->Execute(e);
}

// You can only add one system of each type (class).
```

Event queues:

Events queues can be used by a system to send messages to all systems. The message
type must be a `std::variant` that contains all message types.

```C++
#include <variant>

// Define the message types.
struct EventDialog { std::string msg; };
struct EventKill   { size_t id; };
using EventType = std::variant<EventDialog, EventKill>;

// Create engine passing this type.
using MyEngine = ECS::Engine<System, EventType, MyComponents...>;
MyEngine e;

// Send an event to all systems.
e.Send(EventDialog { "Hello!" });

// In the system, `GetEvents` can be used to read each of the messages in the event queue.
// This will not clear the events from the queue, as other system might want to read it as well.
for (EventDialog const& ev: e.GetEvents<EventDialog>()) {
    // do something with `dialog_ev`...
}

// At the end of each loop, the queue must be cleared.
for(auto& sys: e.Systems())
    sys->Execute(e);
e.ClearQueue();
```

# Component printing

To be able to print a component, the `operator<<` must be implemented. Example:

```C++
struct Direction {
    float angle;
    friend ostream& operator<<(ostream& out, Direction const& dir);
};

ostream& operator<<(ostream& out, Direction const& dir) {
    out << "Direction: " << dir.angle << " rad";
    return out;
}

// Then, to print all components of an entity:
e.Examine(cout, my_entity);

// If you want to print all components of all entities:
e.Examine(cout);

// The result is:
Entity #0:
  - Direction: 50 rad
```

If the `operator<<` is not implemented for a component, the class name will be printed instead.
