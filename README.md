

# Get Started

This lab is design to introduce you to: Writing C++ Helper Libraries, Tile Based logic and 2D coordinate code.
We will be starting this practical from the usual game loop basis.

**Create a new project, add the main.cpp file, and add the new project to CMake.**

## The Goal

The Game we are about to make is a maze game. The player moves a purple circle around from a starting point and tries to get to a determined end point in the shortest amount of time.


The Maze will be loaded from a text file with a super simple format. The logic for loading, rendering, and managing the maze data will be handled by a standalone \"LevelSystem\" library that we will build. This means we can use this code again in future projects. While we are building libraries, we will also make a small Maths helper library to cover some of the functions that SFML doesn't provide with its vector class.

The maze file, along with all other resources can be found in the [Repo (assets/levels/maze_2.txt)](https://github.com/edinburgh-napier/set09121/tree/master/assets)

```
wwwwwwww
wswe   w
w wwww w
w      w
wwwwwwww
```

## Step One - Player Entity

While we won't have any need for complex inheritance or large software patterns, for this practical we are still going to follow the Entity Model. This will be a slight change form Space invaders in that we are not going to inherit from an SFML class.

### Entity.h and Entity.cpp

For this game we will be working with sf::shapes rather than sf::sprites. They are both sibling classes that derive from sf::drawable, but are incompatible with each other (you can't allocate a shape with a sprite.)

Our base Entity class will be abstract, due to having a pure virtual Render() function.

The default constructor is also deleted, meaning the only way to construct it is through the constructor that takes a sf::shape. Meaning that all classes that derive from Entity must provide an sf::shape when they are constructed.

```cpp 
//entity.h
#pragma once

#include <SFML/Graphics.hpp>
#include <memory>

class Entity {
protected:
  std::unique_ptr<sf::Shape> _shape;
  sf::Vector2f _position;
  Entity(std::unique_ptr<sf::Shape> shp);

public:
  Entity() = delete;
  virtual ~Entity() = default;

  virtual void Update(const double dt);
  virtual void Render(sf::RenderWindow &window) const = 0;

  const sf::Vector2f getPosition();
  void setPosition(const sf::Vector2f &pos);
  void move(const sf::Vector2f &pos);
};
```

#### Smart Pointers

An important difference here is that the introduction of smart pointers (std::unique_ptr). So far we have been using "Raw" pointers, and creating them with New(), and deleting them with Delete().

Smart pointers use reference counting to know where a pointer is stored. If it isn't stored anywhere, the reference count is 0 and the smart pointer deletes the memory associated with the pointer. This means that we don't have to call Delete() ourselves, as we would with raw pointers.

This is similar but different to "Garbage Collection" in Java. Smart pointers de-allocates memory when the last reference goes out of scope. This is deterministic, and so you can know when it will happen. With Java, the garbage collection process runs separately and deallocated memory based on internal algorithms -- separate from your code.

### Entity.cpp

The definitions in the .cpp don't do anything fancy. We are just covering the basic movement functions that we no longer have from inhering from sf::sprite. Notice the Update function in particular. This is how we synchronise the state of the entity to SFML.

```cpp
//entity.cpp
#include "entity.h"
using namespace std;
using namespace sf;

const Vector2f Entity::getPosition() { return _position;}

void Entity::setPosition(const Vector2f &pos) { _position = pos;}

void Entity::move(const Vector2f &pos) { _position += pos;}

void Entity::Update(const double dt) {
    _shape->setPosition(_position);
}

Entity::Entity(unique_ptr<Shape> s) : _shape(std::move(s)) {}
```

### Player.h and Player.cpp

Nothing really to see here for the player class, just a standard
implementation of an Entity.

```cpp
//player.h
#pragma once
#include "entity.h"

class Player : public Entity {
 private:
  float _speed;

 public:
  void Update(double dt) override;
  Player();
  void Render(sf::RenderWindow &window) const override;
};
```

For the .cpp, for now we will keep this basic and come back to it!

**You now need to implement code that will move the player around on screen in the Update function.** (Hint: you'll need to define inputs, use the speed, and remember to use dt! You can use the functions from Entity, too)

```cpp
//player.cpp
#include "player.h"
using namespace sf;
using namespace std;

void Player::Update(double dt) {
  //Move in four directions based on keys
  ...
  
  Entity::Update(dt);
}

Player::Player()
    : _speed(200.0f), Entity(make_unique<CircleShape>(25.f)) {
  _shape->setFillColor(Color::Magenta);
  _shape->setOrigin(Vector2f(25.f, 25.f));
}

void Player::Render(sf::RenderWindow &window) const {
  window.draw(*_shape);
}
```

Finally, you will need to create a player object, and link it into your Update and Render code from your game loop in order for it to show!

**REMEMBER: DO NOT USE New(), you should be using Smart Pointers!** (Hint: you should store the Player instance as a unique pointer, and the code already given you will show you how to store and make these!)

{:class="important"}
**At this point you should have a magenta circle moving around a screen. Do not continue on if you haven't got everything working so far.**

Oh, and you can move diagonally... right?


----


# Writing Libraries

For our next piece of work, we are going to write code that we are going to want to use again in future games. Standard practice would be to use good software engineering to isolate all the logic required for our Level loader code to it's own files and minimize coupling between itself and game code.

One approach is to build a "helper class" which is completely separate form the code of the game, but included in the build as headers and .cpp's. We could then easily move this code to a new project by just brining along the needed files. This is fairly common practice and most game programmers have a collection of small "helper" classes that they copy over from project to project.


The next step from this approach is to completely separate the code into a library. We are already using several libraries in our project - all the SFML components. Libraries are code that is complied separately to your program and linked in during the link stage. The primary benefit of this is that we don't have to compile libraries again once they are complied (imagine if we had to build SFML every time we needed to build our game code).


In certain situations you can download libraries pre-compiled from the internet. This isn't great for C++ as the compile settings need to be near identical for your application as for the downloaded library. Furthermore - we can't step down into source code when debugging. Providing a library with a well maintained CMakeLists.txt is by far the best way to distribute your code when building libraries for other programmers to use.

## Static vs dynamic linking -- .lib's and .dll's, Mac: .libs and .dylibs, Linux: .libs and .so files

When we build a library - it will generate a .lib file. Depending on your build settings (Dynamic/Shared linking) it may also produce a .dll/.dylib/.so. The situation is more complicated but the simple explanation is that with dynamic linking the code for our library lives inside the .dll (.so on linux/mac). When our application starts, it loads the code from the .dll into memory. This means that the .dll has to be somewhere the running program can access. We still need to link with a .lib file, this .lib will be an small file which only describes the .dll. Also, this explains the errors you might have seen where a program can't find a .dll file - be careful not to run into this with your final submission!

With static linking, the .lib is compiled into our executable. Meaning we don't need to bring any dlls along. In this case the compiled .lib file contains all the code and will be significantly larger.

## Setting this up

Finding and setting all the right settings in an IDE to set up building and linking libraries is a nightmare. With CMake it's laughably easier (and cross-platform!).

```cmake
## Tile loader lib
file(GLOB_RECURSE SOURCE_FILES lib_tile_level_loader/*.cpp lib_tile_level_loader/*.h)
add_library(lib_tile_level_loader STATIC ${SOURCE_FILES})
target_include_directories(lib_tile_level_loader INTERFACE "${CMAKE_SOURCE_DIR}/lib_tile_level_loader/" )
target_link_libraries(lib_tile_level_loader sfml-graphics)
```

**This should go in the Add External Dependencies section of your CMake file - keeping this file clean makes it much easier to understand and debug!**

The biggest difference here is the call to "add_library" rather than "add_executable" as we'd do with a main project.

From the CMake we can see that we need to put some code in a "lib_tile_level_loader" folder. Library code is exactly the same as wiring any other C++ file, but we don't need a main() function.


## Level System Code

Let's get started with our header.

```cpp 
//LevelSystem.h
#pragma once

#include <SFML/Graphics.hpp>
#include <memory>
#include <string>
#include <vector>
#include <map>

#define ls LevelSystem

class LevelSystem {
public:
 enum TILE { EMPTY, START, END, WALL, ENEMY, WAYPOINT };
        
 static void loadLevelFile(const std::string&,float tileSize=100.f);
 static void Render(sf::RenderWindow &window);
 static sf::Color getColor(TILE t);
 static void setColor(TILE t, sf::Color c);
 //Get Tile at grid coordinate
 static TILE getTile(sf::Vector2ul);
 //Get Screenspace coordinate of tile
 static sf::Vector2f getTilePosition(sf::Vector2ul);
 //get the tile at screenspace pos
 static TILE getTileAt(sf::Vector2f);
 
protected:
 static std::unique_ptr<TILE[]> _tiles; //Internal array of tiles
 static size_t _width; //how many tiles wide is level
 static size_t _height; //how many tile high is level
 static sf::Vector2f _offset; //Screenspace offset of level, when rendered.
 static float _tileSize; //Screenspace size of each tile, when rendered.
 static std::map<TILE, sf::Color> _colours; //color to render each tile type
 
 //array of sfml sprites of each tile
 static std::vector<std::unique_ptr<sf::RectangleShape>> _sprites;  
 //generate the _sprites array
 static void buildSprites();
    
private:
 LevelSystem() = delete;
 ~LevelSystem() = delete;
};
```

**Vector2ul will give you an error, this is something that doesn't exist yet, more on this later.**

That's quite a lot to begin with. Let's pay attention to the public functions first: this is where we declare what our library can do. The protected variables are the internal state that we need for some calculations later on. The whole LevelSystem is a static class, everything is static so we can access everything within it from anywhere (Downside: we can't inherit from it). I've thrown in a handy \#define macro so we can access everything like \"ls::Render()\".

With our LevelSystem.h declared, let's define it in LevelSystem.cpp

```cpp 
//LevelSystem.cpp
#include "LevelSystem.h"
#include <fstream>
#include <iostream>

using namespace std;
using namespace sf;

std::unique_ptr<LevelSystem::TILE[]> LevelSystem::_tiles;
size_t LevelSystem::_width;
size_t LevelSystem::_height;
Vector2f LevelSystem::_offset(0.0f, 0.0f);

float LevelSystem::_tileSize(100.f);
vector<std::unique_ptr<sf::RectangleShape>> LevelSystem::_sprites;

std::map<LevelSystem::TILE, sf::Color> LevelSystem::_colours{ {WALL, Color::White}, {END, Color::Red} };

sf::Color LevelSystem::getColor(LevelSystem::TILE t) {
  auto it = _colours.find(t);
  if (it == _colours.end()) {
    _colours[t] = Color::Transparent;
  }
  return _colours[t];
}

void LevelSystem::setColor(LevelSystem::TILE t, sf::Color c) {
  ...
}

```

We start off by defining all the static member variables declared in the header file. This brings in a C++ data structure that we've not dealt with before, the map. It's statically initialised with two colours, more can be added by the game later. This map is read by the getColor() function which will return a transparent colour if an allocation is not within the map.

**Important: make sure you use find() for maps if you are searching for an element! If you used *auto it = _colours[t]* then we would CREATE the element if it didn't exist! Bad!**

You should complete the setColor function. It's super simple, but you may have to look up the C++ docs/API on the std::map. Again, you should be looking up the API all the time for what you are using. Oh, also note that again we're using smart pointers throughout!

Next up, reading in and parsing the text file.

```cpp 
//LevelSystem.cpp
void LevelSystem::loadLevelFile(const std::string& path, float tileSize) {
  _tileSize = tileSize;
  size_t w = 0, h = 0;
  string buffer;

  // Load in file to buffer
  ifstream f(path);
  if (f.good()) {
    f.seekg(0, std::ios::end);
    buffer.resize(f.tellg());
    f.seekg(0);
    f.read(&buffer[0], buffer.size());
    f.close();
  } else {
    throw string("Couldn't open level file: ") + path;
  }

  std::vector<TILE> temp_tiles;
  for (int i = 0; i < buffer.size(); ++i) {
    const char c = buffer[i];
    switch (c) {
    case 'w':
      temp_tiles.push_back(WALL);
      break;
    case 's':
      temp_tiles.push_back(START);
      break;
    case 'e':
      temp_tiles.push_back(END);
      break;
    case ' ':
      temp_tiles.push_back(EMPTY);
      break;
    case '+':
      temp_tiles.push_back(WAYPOINT);
      break;
    case 'n':
      temp_tiles.push_back(ENEMY);
      break;
    case '\n':      // end of line
      if (w == 0) { // if we haven't written width yet
        w = i;      // set width
      }
      h++; // increment height
      break;
    default:
      cout << c << endl; // Don't know what this tile type is
    }
  }
  if (temp_tiles.size() != (w * h)) {
    throw string("Can't parse level file") + path;
  }
  _tiles = std::make_unique<TILE[]>(w * h);
  _width = w; //set static class vars
  _height = h;
  std::copy(temp_tiles.begin(), temp_tiles.end(), &_tiles[0]);
  cout << "Level " << path << " Loaded. " << w << "x" << h << std::endl;
  buildSprites();
}
```

If we had many more tile types we would switch out that switch statement for a loop of some kind, but for the limited tile types we need; it will do.

The file handling code at the top is nothing special, we read the whole file into a string then close the open file. This may be a bad move if the level file was larger, but this where we can get away with saying "C++ is fast, it doesn't matter".

Once the level string has been parsed into a vector of tile types, an array is created with the final dimensions. Keeping within a vector could be valid, but we don't ever want to change the size of it, so an array seems a better fit.

Notice that while the level file is 2D, we store it in a 1D storage type. If we know the width of the level we can extrapolate a 2D position from the 1D array easily. We do this as C++ doesn't have a native 2D array type. We could create an array of arrays which would do the job, but makes the functions we need to write later slightly more difficult. Also, 2D arrays in most languages are slower to access/serialise/load anyway.

Our level loader library will do more than just parse in a text file, however, it will also render the level with SFML! To do this we will build a list of sf::shapes for each tile in our array. The colour of this shape will depend on the colour association stored in our map. We only need to build this list of shapes once, so this function is called at the end of loadLevelFile().

**You will have to add accessors for height and width at this stage, these will be useful later! You'll need to add them to the header file too**
(This should be pretty easy!)

```cpp 
//LevelSystem.cpp

void LevelSystem::buildSprites() {
  _sprites.clear();
  for (size_t y = 0; y < LevelSystem::getHeight(); ++y) {
    for (size_t x = 0; x < LevelSystem::getWidth(); ++x) {
      auto s = make_unique<RectangleShape>();
      s->setPosition(getTilePosition({x, y}));
      s->setSize(Vector2f(_tileSize, _tileSize));
      s->setFillColor(getColor(getTile({x, y})));
      _sprites.push_back(move(s));
    }
  }
}
```

You need to define getHeight() and getWidth(). we also need yet another function, the getTilePosition().

```cpp 
//LevelSystem.cpp
Vector2f LevelSystem::getTilePosition(Vector2ul p) {
  return (Vector2f(p.x, p.y) * _tileSize);
}
```

As we are just doing maths here we don't need to read into the tile array. We could add some validity checks to make sure the requested tile falls within our bounds, but I like my one-liner too much to bother with that.

There will be times when we need to retrieve the actual tile at a position, both screen-space and grid-space. This is where we must convert 2D coordinates to a single index in our tile array.

```cpp 
//LevelSystem.cpp
LevelSystem::TILE LevelSystem::getTile(Vector2ul p) {
  if (p.x > _width || p.y > _height) {
    throw string("Tile out of range: ") + to_string(p.x) + "," + to_string(p.y) + ")";
  }
  return _tiles[(p.y * _width) + p.x];
}
```

Most of this function is taken up by a range check (Where we throw an exception like we have been earlier). The real calculation is in that last line. The secret is to multiply the Y coordinate by the tile map width, and add the X.

**Don't continue on unless you understand why and how this works, it gets more difficult from here on out. Remember to ask for help if you need it!**

Doing the same, but with a screen-space coordinate is not any different. However as we are dealing with floats now, we must check it's a positive number first, then we can convert to grid-space, and call our above function. Again, don't continue on unless you understand why and how this works.

```cpp 
//LevelSystem.cpp
LevelSystem::TILE LevelSystem::getTileAt(Vector2f v) {
  auto a = v - _offset;
  if (a.x < 0 || a.y < 0) {
    throw string("Tile out of range ");
  }
  return getTile(Vector2ul((v - _offset) / (_tileSize)));
}
```

And finally - here lies our Render Function. Nice and Simple.

```cpp 
//LevelSystem.cpp
void LevelSystem::Render(RenderWindow &window) {
  for (size_t i = 0; i < _width * _height; ++i) {
    window.draw(*_sprites[i]);
  }
}
```

## Linking our Library

While the library can build by itself, that's rather useless to us. We need to link it into our lab code. Back to CMake:

```cmake
target_link_libraries(... lib_tile_level_loader sfml-graphics)
```

You probably could have guessed this addition. Just add the library target name to your link_libraries, just like we do with SFML. You can add as many as you like, separated by spaces.


## Maths Library


Remember that Vector2ul type that doesn't exist? Before we can use this, we need to fix that error!

The vector maths functionality of SFML is quite lacking when compared to larger libraries like GLM.
We could bring in GLM and write converter functions to allow it to interface with SFML.

This would be a good idea if we needed to advanced thing like quaternions, but we don't.

Instead we will build a small add-on helper library to add in the functions that SFML misses out.

### CMake

Firstly, like we did with our first library, add this to CMake in Add External Dependencies section:

``` cmake
# Maths lib
add_library(lib_maths INTERFACE)
target_sources(lib_maths INTERFACE "${CMAKE_SOURCE_DIR}/lib_maths/maths.h")
target_include_directories(lib_maths INTERFACE "${CMAKE_SOURCE_DIR}/lib_maths" SYSTEM INTERFACE ${SFML_INCS})
```

This is slightly different to the level system library. This time we declare the library as INTERFACE.

This changes some complex library and linker options that are beyond the scope of explanation here. The simplest explanation is an INTERFACE library target does not directly create build output, though it may have properties set on it and it may be installed, exported and imported. Meaning that in Visual Studio the library will look like it is part of our main lab code (except it isn't. Magic.)


There are many different way to create and link libraries, CMake allows us to change these options from a central point and not worry about digging through IDE options.


We need to link the maths library, against the LevelSystem library. edit the level systems cmake code to include lib_maths


```cmake
target_link_libraries(... lib_tile_level_loader lib_maths sfml-graphics)
```

As this is a static library - and doesn't produce a compiled output, everything will be in a header.

**Go create the correctly named filed in the correct folder - you should be able to work it out from the CMake code above!**

If you can't see the file after recompiling CMake, you've probably forgotten to add it to your tile map project in the target_link_libraries link correctly. If done correctly the file will appear in the header file folder IN your project, rather than as a separate project - that's because we used INTERFACE this time. Bear in mind, however, that this is the same file across multiple projects, so changing it in one place will change it all in places!

To extend the functionality of sf::vectors we must first be within the same namespace. From here we can define code as if we were inside the SFML library code itself. Things get a little strange if we want to change or override functions that already exist, but we don't here as we are only creating new functionality. As such, don't worry about it for now!

We start by creating a new vector type the 'Vector2ul' which will use size_t (i.e the largest unsigned integer type supported on the system) as the internal components. We will use this for the tile array coordinates.

From this we implement the standard vector maths functions that any self respecting game engine would have. Length, Normalization, and Rotation. I've left these incomplete so you wil have to dredge up your vector maths skills to complete them.

Lastly, we override the << stream operator to us do things like "cout << vector". Useful for debugging.

```cpp
//maths.h
#pragma once

#include <SFML/System.hpp>
#include <cmath>
#include <iostream>
#include <vector>

namespace sf {
  //Create a definition for a sf::vector using size_t types
  typedef Vector2<size_t> Vector2ul;
  // Returns the length of a sf::vector
  template <typename T> double length(const Vector2<T> &v) {
    return sqrt(...);
  }
  // return normalized sf::vector
  template <typename T> Vector2<T> normalize(const Vector2<T> &v) {
    Vector2<T> vector;
    double l = length(v);
    if (l != 0) {
      vector.x = ...
      vector.y = ...
    }
    return vector;
  }
  //Allow casting from one sf::vetor internal type to another
  template <typename T, typename U>
  Vector2<T> Vcast(const Vector2<U> &v) {
    return Vector2<T>(static_cast<T>(v.x), static_cast<T>(v.y));
  };
  // Degreess to radians conversion
  static double deg2rad(double degrees) {
    return ...
  }
  //Rotate a sf::vector by an angle(degrees)
  template <typename T>
  Vector2<T> rotate(const Vector2<T> &v, const double degrees) {
    const double theta = deg2rad(degrees);
    const double cs = cos(theta);
    const double sn = sin(theta);
    return {(T)(v.x * cs - v.y * sn), (T)(v.x * sn + v.y * cs)};
  }
  //Allow sf::vectors to be cout'ed
  template <typename T>
  std::ostream &operator<<(std::ostream &os, const Vector2<T> &v) {
     os << '(' << v.x << ',' << v.y << ')';
     return os;
  }
} 
```

Once you've fixed all of the bugs, you should be good to use your new vector library!

**Include this library in your LevelSystem.h file, and make sure everything now compiles! You will need to use a relative path.**

## Using the library

Give it a test, Call some library functions from your lab code.

```cpp
\\main.cpp
#include "LevelSystem.h"

...

void load() {
  ...
  ls::loadLevelFile("res/levels/maze.txt");

  // Print the level to the console
  for (size_t y = 0; y < ls::getHeight(); ++y) {
    for (size_t x = 0; x < ls::getWidth(); ++x) {
      cout << ls::getTile({x, y});
    }
    cout << endl;
  }
}
...
void Render(RenderWindow &window) {
  ls::Render(window);
  ...
}
```

That should be all we need to successfully build both our libraries and our game. Give it a go. Build. Run. See if your hard work typing all this has paid off.

**Oh no, did it throw an error about the tile map?** Look at the code for how we test it is all loaded: we only count a line as done when it ends with a new line... can you figure out what is likely missing from your map file?

I know it was lots of work to get here, but you can use loads of what you've just created for your own games!

{:class="important"}
**Your code should compile and your game should run. Congratulations if you've got here! Make sure to show us!**


## Making the Game a Game


### Player Class

Disallow the player from moving into a tile:

Hint:

``` cpp
bool validmove(Vector2f pos) {
  return (ls::getTileAt(pos) != ls::WALL);
}
```

### Advanced tasks


- Start the Player from the "Start Tile"
- End the game When the player hit's the end tile
- Time how long it takes to complete the maze
- Show current, previous and best times

---

