---
title: "Dungeon Rooms Mod v3.0 - How does it work?"
summary: DRM v3.0 entirely relies on a combination of raytracing (to get visible blocks within FOV), the hotbar map, the player's coordinates, and a ton of math to detect rooms and place waypoints. Read the full post for more details.
weight: 2
---

## Background Info
 In order to detect rooms and place waypoints, Dungeon Rooms Mod 3.0 (DRM v3.0) relies entirely on a combination of raytracing (to get visible blocks within FOV), the hotbar map, the player's coordinates, and a ton of math.


### Raytracing (Raycasting):
 To be specific, the mod uses raycasting, which is sort of a form of raytracing, to obtain visible blocks. Raycasting is essentially raytracing except we don't care about any secondary sources of rays (e.g. the sun). When DRM v3.0 detects a player has entered a new room, it sends out 1024 raycast vectors (sounds like a lot, but it really isn't, taking less than a few milliseconds to calculate) which are contained within the player FOV. DRM v3.0 saves the first object that the raycast hits. This means that **at no point** is a block outside of the player's FOV ever used by DRM v3.0 and that all blocks which are used are blocks that the player can see with their own eyes. 

 This represents a significant deviation from the standard method of finding blocks that most Skyblock mods like SBE, DSM, NEU, and Skytils use. Those mods scan and iterate through every single block in a radius around the player, not caring whether they are visible to the player or not, meaning that, in the process, thousands of non-visible blocks are scanned. In most cases this is fine, but mods like SBE scan with a radius of 25 blocks specifically for blocks that are not visible to the player, namely piston heads. In the SBE's developer's own words, scanning non-visible blocks like this is "quite literally on the same level as X-ray." So, not wanting DRM to be literally x-ray, raycasts are used instead to get visible blocks, so that non-visible blocks are never even accessed by the mod. Meanwhile, mods like SBE (as of their version 2.1.2) still use the "literal x-ray" method.

### Hotbar Map:
 DRM v3.0 also uses the hotbar map to help with room detection. The hotbar map is the map which is given to you after starting a dungeon run in slot 9 of your inventory. From now forward, the "hotbar map" will refer to the map in slot 9, while the "physical map" will refer to the actual blocks constituting the world. DRM v3.0 pulls data from the hotbar map once every time a player enters into a new room. The player's map does not have to be open at that point. This is allowed in the same way mods are allowed to place the hotbar map on the screen and update it without the player having to look at the map. 

 The hotbar map serves three main purposes.
 1. To get the color of the room the player is in.
 2. To get the size of the room the player is in.
 3. To get the orientation (or possible orientations) of the room the player is in.

### Player Coordinates:
 Player coordinates are the x, y, and z values which a person can find in the F3 menu at all times, and are also displayed by many other mods. Almost all major Skyblock mods use player coordinates for one thing or another. DRM v3.0 primarily uses player coordinates to determine whether the player has left the boundaries of a room.

### A Ton of Math:
 The rest of this article along with the [source code on Github](https://github.com/Quantizr/DungeonRoomsMod/) will hopefully help you understand the math used by this mod.


---


## Room Detection:
 I had a whole several thousand-word explanation of how the mod works, then I realized no one was gonna read it... so here's the shorter version. At the start of the dungeon run, the mod saves the location of the closest physical corner and closest hotbar map corner (all done using math, no block scanning). From there, coordinates can be translated between physical and map coordinates since dungeons operate on a 32 block wide grid and the width of rooms on the hotbar map can easily be read.

 The mod scans the hotbar map for connected rooms, then translates the coordinates of the connected hotbar map rooms to physical rooms. The mod checks every half a second to see if a player is still within the bounds of the physical rooms. When a player leaves the old coordinates, the mod scans the hotbar map, checking the size and color of the room. If the room is an entrance, miniboss, fairy, or blood room, which is determined based on color on the hotbar map, that room name is displayed on screen. Otherwise, the mod initiates a raytrace scan, sending out 1024 vectors spread out across the player's FOV. The raytrace returns the first block which the vector hits, and that block's location and block type is saved into the mod's memory. The mod then filters out duplicate blocks (where multiple vectors hit the same block) and filters out other blocks which are not contained in the ".skeleton" room data files.

 The mod then finds the possible rotations of the room, since dungeon rooms can be rotated in multiple directions. The direction of some rooms, like L-shape rooms, can be instantly determined because their shape appears different on the map depending on their rotation. Rectangular rooms have two possible rotations. Square rooms require all four directions to be checked. The mod loads in a list of rooms of the same size as the room the player is currently in then iteerates through the blocks which were previously saved, checking each possible rotation direction, until only one room remains. The mod checks another 10 blocks to be sure it isn't a missing room with similar blocks. If a singular room remains, that room name is displayed on the screen. Otherwise, if no matching rooms are found, the mod waits a few seconds before initiating another raytrace. This whole process takes less than a few milliseconds and is only run once every time the player enters a room, so it causes no noticeable lag at all.

 Once the mod knows the name of the room and the orientation, waypoints can be displayed. Waypoints are always at the same relative coordinates, so it is a simple problem of converting the relative coordinates to the actual coordinates, which can be done entirely with relatively simple math.

 While this all sounds complicated, it is, more or less, what your brain does when it sees a dungeon room. Your eyes see different blocks in the map, and your brain compares them with rooms you already know. Once your brain figures out what room you're in, assuming you've memorized secrets, your brain can then tell you where the secrets are. This is what this mod does in code form. Unlike other solvers, such as experimentation table solvers (>99% of humans cannot memorize the entire pattern of the experimentation table), memorizing and finding secrets is not beyond the ability of the average human, so comparatively, this mod arguably does not provide an unfair advantage over the average player.

 If you want to know exactly how everything works, just read [the code](https://github.com/Quantizr/DungeonRoomsMod).


---


## Room Data (.skeleton files)
 This mod compares the relative coordinates of the raycasted blocks with coordinates found in the room data files, which are the .skeleton files in the resources directory of the mod. (They are called .skeleton files because they only store the main structural blocks of the room and don't contain most decorative blocks, essentially creating a "skeleton" of the room. Also, skeletons are a Minecraft mob and this is a Minecraft mod so... yeah) There is one skeleton file for each possible room in the dungeon.

 After many, many iterations, the current format of the .skeleton file is a Java serialized array of ordered primitive longs. This was determined as the most efficient method, in terms of both storage and RAM, to store the more than 5 million blocks contained within all the files combined. Other versions included using a HashMap for each room (used ~400MB additional RAM), using a custom implementation of a HashMap made by TimeAndSpaceIO known as SmoothieMap (used ~135MB additional RAM), storing the blocks as an ArrayList of object Longs (used ~125 MB additional RAM), along with several other implementations. After around a dozen hours of trial and error, it turned out that one of the simplest solutions, using an array of ordered primitive longs, was the best solution (used ~40 MB additional RAM). Of course, this came with the disadvantage of not being able to use HashMap's O(1) contains function, but in real world usage, the O(log n) speed of using a binary search on the array wasn't that much slower, even if some of the arrays contained more than 100,000 values.

 You might be wondering, wait, how did you fit all the block data into a long (8 bytes)? The x, y, and z coordinates, along with the custom block id value were each converted to a short (2 bytes), then combined into a long. The custom id value consists of the block id multiplied by 100 with the metadata id of the item equivalent of the block added on. For example, the custom block id for Stone is 100, and the custom id for Black Stained Clay is 15915. Of course, a short has a maximum value of 32,767, meaning that any block with a Minecraft id of over 327 would not be able to be stored. This is not a problem in the case of this mod because the highest value stored is that of Black Stained Clay (15915).

 The four shorts are placed together into a unique long with this function (just a lot of bit manipulation):
 ~~~ java
  shortToLong(short a, short b, short c, short d) {
    return ((long)((a << 16) | (b & 0xFFFF)) << 32) | (((c << 16) | (d & 0xFFFF)) & 0xFFFFFFFFL);
  }
 ~~~
 *(this might not have been the most efficient bit manipulation, but it works fast enough)*


---

 **Dungeon Rooms Mod Links:**

 Discord: https://discord.gg/kr2M7WutgJ

 Github: https://github.com/Quantizr/DungeonRoomsMod

---