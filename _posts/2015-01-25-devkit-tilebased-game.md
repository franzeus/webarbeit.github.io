---
layout: post
title: Devkit and Easystar - How to build a tile-based game
description: Tutorial, learn how to build a tile based game in devkit gameclosure
---

<div class="message">
In this post you will learn the basics to create a tile based game with the <a href="http://gameclosure.com">game{closure}</a> gaming framework.
</div>

<div class="message message_warning">
<h4>Yes you! Read this:</h4>
You should already know what a tile based game is and know the basics about the gameclousure framework to understand this tutorial! (<a href="http://www.tonypa.pri.ee/tbw/tut00.html" target="_blank">Why tilebased games</a>)
</div>

## What are we building?

<img src="/public/posts/devkit_tilebased_game.png" alt="Devkit a tile based game" class="image_responsive">

We will build the very basics of a tile-based game. That means:

* Generate tiles, based on a 2D-Array map
* Tiles will consist of floor, wall and moving tiles (e.g. a players)
* On tap event, the player-tile moves to another tile (Path finding with [Easystar](http://www.easystarjs.com/))

## Files & Objects

The easiest way to get an overview about the files is to explore the [github repo](http://github.com/)

However, in our `/src` folder we have:

* `Application.js` - To setup the game and views
* `/views/GameView.js` - "The world" which contains all the tile-views
* `/models/Tile.js` - Basic tile object (`ui.View`)
* `/models/MoveTile.js` - Extends Tile.js, can change its position

## Generate the tiles - GameView.js

The GameView.js is a `ui.View` and covers the entire screen. It is the parent ("container") view of all the other views - like the tiles.

### First there was a 2D-Array

So basically we define our world (columns and rows) in a 2D-Array, process the array and add child-views to the GameView.

<img src="/public/posts/devkit_uml.png" class="image_responsive">

Each element in our 2D-Array represents a tile. Where a `1` is a wall (non-walkable) and a `0` a floor-tile (walkable).

(`/src/views/GameView.js`)

```javascript
// @var {Number} - The size in pixel of one tile (width x height)
var TILE_SIZE = 50;

/*
  0 = Floor
  1 = Wall
*/
var map = [
  [1, 1, 1, 1, 1, 1, 1, 1],
  [1, 0, 0, 0, 0, 1, 0, 1],
  [1, 0, 0, 0, 0, 0, 0, 1],
  [1, 0, 0, 0, 0, 0, 0, 1],
  [1, 0, 0, 0, 0, 1, 0, 1],
  [1, 0, 0, 0, 0, 1, 0, 1],
  [1, 1, 1, 1, 1, 1, 1, 1]
]
```

The size of one tile is set in `TILE_SIZE` to 50 pixel. This means that each element/tile will take up 50x50 pixel in our game.

<div class="message message_info">
<h4>Why 50px?</h4>
<i>An MIT Touch Lab study of Human Fingertips to investigate the Mechanics of Tactile Sense found that the average width of the index finger is 1.6 to 2 cm (16 – 20 mm) for most adults. This converts to 45 – 57 pixels, [...]</i>
<a href="http://www.smashingmagazine.com/2012/02/21/finger-friendly-design-ideal-mobile-touchscreen-target-sizes/">Smashingmagazine</a>
</div>

To create the Tile-Views we iterate over the 2D-Array, create `new Tile()` objects and push them to the GameView child-views with `this.addSubview(tile)`.

```javascript
this.addTiles = function(map) {
  var tilesize = TILE_SIZE;
  var x = 0;
  var y = 0;

  for (var i = 0; i < map.length; i++) {

    for (var k = 0; k < map[i].length; k++) {
      var type = map[i][k];
      var tile = new Tile({//...});

      // Add tile-view as child view of the GameView
      this.addSubview(tile);

      // Bind tap event for each tile
      tile.on('Tile:tapped', bind(this, function(tile_entity) {
        this.moveTileTo(active_tile, tile_entity);
      }));
    // ...
      x += tilesize;
    }
    y += tilesize;
    x = 0;
  }
```

### Add the player tile

Our player-tile is a `MoveTile` object. `MoveTile` objects inherit from the `Tile` object and contain some
properties and methods to move around:

```javascript
// Speed of the tile
this.speed = 40;

// Basically a queue or list of waypoints (indexes of tiles)
// E.g. [ {x: 1 y: 2}, { x: 1, y: 3}, ... ]
this.move_queue = [];
```

When a user taps on a Floor-tile, the player should move there by using a pathfinding algorithm (we will use [Easystar](http://www.easystarjs.com/)).
Thats why we use a queue data structure to store all the tiles the moving-tile has to visit.

<img src="/public/posts/devkit_queue.png" class="image_responsive">

To add the player tile to the GameView we simply create a `new MoveTile()` and again add it with `this.addSubview()` to the parent GameView. Notice: the position of the player is still hard coded and the best practice would of course be to have it already in our `map` 2D-array.

```javascript
// GameView.js
this.addPlayer = function() {
  player = new MoveTile({
    col: 2, // start col
    row: 2, // start row
    x: TILE_SIZE * 2,
    y: TILE_SIZE * 2,
    width: TILE_SIZE,
    height: TILE_SIZE
  });
  this.addSubview(player);
  this.setActiveTile(player);
  // ...
```

This should happen after we build the map, so that the player-tile is in front and not hidden behind a floor or wall tile.

## Move player-tile to a tile

When we parsed our map in `this.addTiles()` and added our tiles, we also binded a `'Tile:tapped'` event to each tile. The following will happen, when a user taps on a Floor-tile:

1. Tap on a Floor-Tile
2. `Tile::tapped` event is triggered
3. `GameView::moveTileTo(source, target)` gets called
4. "Easystar magic" and calculating the `path`
5. Setting queue of player tile `MoveTile::setQueue(path)`
6. Start working up the `move_queue` of the player tile

<div class="message message_warning">
  <h4>What about the collision detection?</h4>
  This is what Easystar already does for us. We can define that a 1 is a wall and not walkable. So Easystar will calculate the path and avoid all walls etc and return us the correct path we need.
</div>

So for each element in the `move_queue` the `MoveTile::moveTo()` method gets called and we animate the player-tile view with the build-in `animator` method:

```javascript
// @param {Object} Point - x, y coordinates to go to
this.moveTo = function(point) {
  if (!point) throw "Point param missing";
  var currentX = this.style.x;
  var currentY = this.style.y;
  var targetX = point.x;
  var targetY = point.y;
  // Basic v = s / t - to know how much time this tile needs
  // to get to the target.
  var line = new Line(currentX, currentY, targetX, targetY);
  var distance = line.getLength();
  // (we should actually use this.move_queue.length * tile-size)
  var t = Math.round((distance / this.speed) * 100);
  return this.animator.now({ x: targetX, y: targetY }, t, animate.linear);
};
```

## Whats next?

Of course this is just the very basic setup for a tile-based game.
The next steps would be to create a `FloorTile`, `WallTile`, `PlayerTile` and have a factory method to add them in the `addTiles()` method pending on the number in the 2D-Array.

We also didn't update the map when the player moves. Which is also important when we have more moving tiles and want Easystar to correctly calculate the path.

