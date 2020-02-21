# Valhalla Gurka Tests

Tesst in this directory are "integration" level tests.  They check the full pipeline from
map parsing through to route/match/etc result generation.

# Test structure

Inside the gurka/ directory, every file called `test_*.cc` is built as a distinct executable.
A file called `test_turns.cc` will end up as `build/test/gurka/gurka_turns`

# Using Gurka

The `gurka.h` header provides helper functions for generating small test maps from ASCII art,
building tiles from them, executing Valhalla API requests against those tiles, and helpers
for checking parts of the API response for expected results.

A minimal example test looks like this:

  1. Create a file called `test/gurka/test_example.cc`
  2. Populate a test case

```cpp
#include <gtest/gtest.h>
#include "gurka.h"

TEST(TestSuite, TestName) {

    const std::string &ascii_map = R"(
        A----B----C
    )";
    const gurka::ways ways = {{"ABC", {{"highway", "primary"}}}};

    auto map = gurka::buildtiles(ascii_map, 100, ways, {}, {}, "test/data/example");

    auto result = gurka::route(map, "A", "C", "auto");

    EXPECT_EQ(result.directions().routes_size(), 1);
    EXPECT_EQ(result.directions().routes(0).legs_size(), 1);
}
```

To build it, you can do `make gurka_example`, and `make run-gurka_example`.

You can build and execute gurka tests with `make run-gurka`

# How to construct a map

You need 4 things to build a test map.  It's helpful to understand the [OSM Data Model](https://labs.mapbox.com/mapping/osm-data-model/) before beginning.

  1. Define the positions of nodes on the map by drawing an ASCII representation.
     The map is interpreted as a 2d grid (see `gridsize` below). The letters A-Za-z0-9
     represent OSM nodes.  Only node positions are interpreted from the ASCII grid,
     but you can use any other character to help give context (i.e. use ---- to indicate
     that two nodes will be connected by a way).
     Using C++ Raw String Literals (6) https://en.cppreference.com/w/cpp/language/string_literal
     is an easy way to to draw multiline string maps.
     Example:

```cpp
const std::string ascii_map = R"(
    A---B---C     j
        |        /
    D---E---f---h
)";
```

     **Remember**: only the position of the A-Za-z0-9 characters matter - other characters
     are ignored and are just helpful for describing your intent.

  2. Define how the nodes are connected together (as OSM ways).  You need to build a
     `gurka::ways` object.  This object has a `std::string` key which defines a sequence
     of nodes that are connected, then a `std::vector` of key/value pairs that describe
     the tags to go on a way.  C++ inializer-list syntax allows for a fairly compact
     expression of a list of ways and their tags.
     Example:

```cpp
const gurka::ways ways = {
    { "ABC",   // String referencing nodes to be connected in order
      { {"highway","motorway"},  // key/value tags to be put on the way
        {"access","none"},
        {"name","Test Road"} }
    },
    { "BE",  
      { {"highway","motorway"},
        {"access","none"},
        {"name","Test Connector"} }
    },
    ...
}
```

  3. Define properties on the nodes, if any.  Here, you need a `gurka::nodes` object.  Construction
     is identical to the `gurka::ways` object, except that the primary key should just be
     a single character (not a `char`, but a 1-character `std::string`).

```cpp
const gurka::nodes nodes = {
    { "A",   // String referencing nodes to be connected in order
      { {"barrier","true"} } // key/value tags to put on the node
    },
    ...
}
```

  4. A list of relations.  A common example is a [restriction relation](https://wiki.openstreetmap.org/wiki/Relation:restriction), which restricts possible maneuvers.
     You need to build a `gurka::relation` object.  Construction here is a little more
     complex, but C++ initializer list syntax still works (get used to lots of {}).


```cpp
const gurka::relations relations = {
    {
        { // List of the members on the relation
            gurka::relation_member{gurka::way_member, "kh", "from"},
            gurka::relation_member{gurka::way_member, "il", "to"},
            gurka::relation_member{gurka::way_member, "hi", "via"}
        },
        { // List of the key/value tags on the relation
            {"type", "restriction"},
            {"restriction", "no_right_turn"}
        }
    },
    ...
};
```

Once you have all these elements, you can now use them to build an OSM PBF file, and generate
tiles for it, using:

```cpp
    auto map = gurka::buildtiles(ascii_map, 100, ways, nodes, relations, "test/data/example");
```

The `map.pbf` and tiles will be written to `test/data/example`.  The value `100` is the size of the grid to be used.  It's in metres, and decides the distance between the nodes in the ascii map.

The returned `map` object has two properties:

  - `config` - the boost::property_tree that was used to generate the tiles
  - `nodes` - an `std::unordered_map` that with keys for the nodes, and the values are the PointLL that was assigned (this is useful for getting the coordinate of a node in a generated map)


# Using a generated map

Once you have map tiles, you can then do tests on them.  Gurka provides some helpers to make
it easy to perform some normal interactions.

```cpp
valhalla::Api
route(const gurka::map& map, const std::vector<std::string>& waypoints, const std::string& costing) {
```

This performs a valhalla route request on the map.  You can use named waypoints that you've drawn
on the map as positions to route between.


# Utility functions

The main purpose of Gurka is to write high-level, end-to-end tests on minimaps.  There are some
low-level helper functions available in case you want to do something a little more custom:

  - `gurka::detail::build_config(workdir);` - builds a `boost::property_tree` for tile generation in `workdir`
  - `gurka::detail::map_to_coordinates(ascii_map, gridsize);` - calculates coordinates for all the A-Za-z0-9 nodes in the `ascii_map` given the `gridsize`
  - `gurka::detail::build_pbf(node_locations, ways, nodes, relations, pbf_filename);` - generates an OSM PBF for the nodes, ways, and relations you've defined.  The `nodemap` is the result of `gurka::detail::map_to_coordinates`