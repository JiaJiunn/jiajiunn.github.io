---
name: Multiplayer 3D Game Engine
tools: [OpenGL, Java, sockets]
image: /assets/game_engine/game_engine.png
description: A simple 3D game engine written in Java using OpenGL, with multiplayer support via sockets.
# external_url: https://github.com/JiaJiunn/multiplayer-3d-game-engine
---

# Implementing a Multiplayer 3D Game Engine

You can find the project implementation [here](https://github.com/JiaJiunn/multiplayer-3d-game-engine).

<br />

Recently, I've been intrigued by the idea of implementing my own custom game engine. I've previously done some work on game software before (available for download [here](http://en-ci-gdiac.coecis.cornell.edu/gallery/nite_bite/)), but back then, I mainly worked on a single-player 2D game, where a lot of the challenges came with tuning the game's levels, bots, and difficulty, and making sure that overall the game is well-balanced. 

All those are, of course, necessary for a game to be well-received -- but this time, I wanted to explore the technical challenges that come with simply making a multiplayer 3D game, which is what this project is about. Since my aim is to learn more about 3D graphics and socket programming, here my end goal isn't a well-balanced game, rather just a game engine that can support multiple players in a 3D game environment.

## The 3D Game Engine

I've never dabbled with 3D games before, but looking through online forums, I decided to build my game engine off OpenGL (rather than customizable engines like Unity and Unreal Engine). This would allow me to build a custom engine from scratch, rather than spending time learning and customizing an existing one. Coincidentally, a YouTuber whose devlogs I follow, ThinMatrix, seems to also have a rather comprehensive [online series](https://www.youtube.com/watch?v=VS8wlS9hF8E&list=PLRIWtICgwaX0u7Rf9zkZhLoLuZVfUksDP) on OpenGL and its components, which is where I started off with.

Looking through his tutorials, I ended up with a system structured as follows:

<br />
![](/assets/game_engine/all.png)
<br />
where each box represents a class.

To break things down step by step:

1. I first created a DisplayManager, which handles everything related to creating, updating, and destroying the game window.
2. Each 3D model is represented by a TexturedModel, which is a combination of a RawModel (which accesses the VAO storing all its vertex and normal information) and ModelTexture, which stores the textures of the model. Ideally, this would allow for models to swap textures, such as when players want to change skins and such. 

<br />
![](/assets/game_engine/player.png)
<br />

{:start="3"}
3. Each instance of a TexturedModel is called an Entity. This is just a TexturedModel with pose information. For instance, to create a forest, we would create a tree TexturedModel (or perhaps several different species of trees), and spawn multiple Entities, where each tree is a Entity with a certain position and orientation.
4. A Player is basically an Entity that is movable. It inherits from Entity, and additionally checks for user movement inputs, which it responds to accordingly.
5. I also created Camera and Light classes, which handles the camera view and lighting respectively. Note that Camera takes in the Player position at any point in time to calculate the camera position, such that it maintains a 3rd person view at all times.
6. We handle terrains seperately by representing the terrain in a Terrain class. The terrain consists of a RawModel, which controls the landscape itself, and a TerrainTexture, which handles the texture of the landscape. I use a blend map to allow blending multiple textures in a single terrain, such as having pavament that cuts across the grassy land or have patches of dessert. More in-depth explanations on the depth map can be found [here](https://www.youtube.com/watch?v=-kbal7aGUpk&list=PLRIWtICgwaX0u7Rf9zkZhLoLuZVfUksDP&index=17).

<br />
![](/assets/game_engine/terrain.png)
<br />

{:start="7"}
7. The loaders package, which include the OBJLoader and Loader, handle loading OBJ and texture files into formats compatible to be stored in VBOs. An in-depth explanation of the pre-processing can be found [here](https://www.youtube.com/watch?v=KMWUjNE0fYI&list=PLRIWtICgwaX0u7Rf9zkZhLoLuZVfUksDP&index=9).
8. Next, I implemented the vertex and fragment shaders, as explained in the video [here](https://www.youtube.com/watch?v=AyNZG_mqGVE&list=PLRIWtICgwaX0u7Rf9zkZhLoLuZVfUksDP&index=4). 

<br />
![](/assets/game_engine/renderer.png)
<br />

These shaders are under the abstract class ShaderProgram, in classes StaticShader and TerrainShader, where we handle shading for the entities and terrain separately. Lastly, the entities and terrain are renderered (i.e. drawed) by the EntityRenderer and TerrainRenderer respectively. All of the shader and rendering code lives under the MasterRenderer, which is in charge of all the rendering based on updated object information.

## Multiplayer Support

For implementing multiplayer support, I turned to socket programming. Having no prior experience working with sockets and networking, I found [this](https://docs.oracle.com/javase/tutorial/networking/sockets/clientServer.html) tutorial really helpful in clearing up the details needed for settin up a simple server-client socket. I treated the server as a way to store some state of all the in-game players (and in the future, details on the environment such as where the lampposts and trees are. Currently the landscape is just bare). The client, i.e. players, then ping the server at a certain frequency to get updated information on other players, as well as updates the server on the current player's information.

I started off by storing the position and orientation of each player in the server. Expectedly, this led to really terrible player movements, as other players' positions only get updated at whatever frequency the ping is. Hence, instead, I store the current speed and turning speed of each player (as well as whether the player initialized a jump). This allows for the player's position to be updated by using the previous ping's speeds, hence allows for smoother movement between pings.

Implementation-wise, within the server, I simply create a HashMap `pos_database`, which stores the current player speed, turning speed, and jump, where the key is each player's ID, represented below by the client ID.

<br />
![](/assets/game_engine/server.png)
<br />

Hence, at every ping, the client (player) would update the player's own speed, turning speed, and jump based on the player's input. The server would then return the most recent version of `pos_database`, which contains the speed, turning speed, and jump information of all other players in the server.

## Remarks

<figure class="video_container" align="center">
  <video controls="true" width="700" allowfullscreen="true">
    <source src="assets/game_engine/AppDemo.mp4" type="video/mp4">
  </video>
</figure>

Now, we have a really simple 3D game engine to build off of! This game engine is still far from being completed, of course, and there are plenty of optimizations that can be made to the server-client messages as well as the physics engines themselves (collisions between entities are still not implemented for instance). Nevertheless, I had a lot of fun learning and working on this project, and I hope you got something out of this too!