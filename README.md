# 🌍 Digital Civilization Simulator (wannabe)

The program generates a two-dimensional elevation grid using Perlin noise. This grid is then converted into individual tiles – currently, only two types are available: water and land.

On the generated map, plants, animals, and agents are placed randomly. Each organism instance is created based on its own prefab. Organisms have a transform object, split into a readonly and a writer part, which is responsible for their position in the world.

Additionally, animals and agents are equipped with brains. Each brain relies on a perception, which is a fragment of data extracted from the world. Perception is represented by a Perception object, consisting of several arrays. Each record corresponds to a single map field and is broken down into arrays such as x[], y[], and organism_id[].

Agents can follow different strategies. Currently, two are implemented: moving in random directions and attacking animals within range.

The world is divided into two main layers:

world_map – responsible for the layout of tiles,

world_state_service – stores organisms and their positions.

Rendering is performed based on snapshots of the game state. A snapshot is an object with arrays containing encoded information about the world (similar to perception). Then, world_presenter decodes the snapshot, selects the appropriate sprites, and displays them in the correct positions on the screen.

The entire program runs synchronously. Initially, I experimented with asynchronous implementation (separately for each organism), but due to performance issues, I switched back to a synchronous model. The main game loop operates in time intervals (ticks) and propagates signals down to the organisms.

Organism movement consists of a sequence of actions executed step by step across subsequent ticks. AnimalMovement passes a list of actions to the SequenceAction class. For animals, these are: first a rotation toward the movement direction, followed by position interpolation using an offset. After the movement is completed, the organism’s new position is set and passed to the world_state_service via the event_bus and world_interactions_handler. Then, world_state_service publishes an event (also through the event_bus), informing all organisms of the updated position.

The project was originally intended to be much more ambitious – I planned to observe interactions between agents, assign them a set of traits and life parameters. However, I underestimated the time required to build a game engine and did not fully account for performance challenges. In practice, it became a constant struggle between code readability and efficiency. For this reason, I decided to conclude the project at its current stage.



![DEMO](https://github.com/user-attachments/assets/437c7bbf-7313-46e5-969b-d2213efba55f)
