The Spimbot Documentation
=========================

Version **1.1.3**  
Last updated **March 8, 2024**

Table of Contents
=================

* [1 The Game](#game)
    * [1.1 The Map](#map)
    * [1.2 The HUD](#hud)
    * [1.3 Tile Types](#tiles)
    * [1.4 The Objective](#objective)
    * [1.5 Scoring Points](#points)
    * [1.6 Resetting](#reset)
    * [1.7 The SPIMBot](#bots)
* [2 SPIMBot MMIO](#mmio)
    * [2.1 Orientation Control](#orientation)
    * [2.2 Odometry](#odometry)
    * [2.3 Bonk (Wall Collision) Sensor](#bonk)
    * [2.4 Timer](#timer)
    * [2.5 Puzzle](#puzzle)
        * [2.5.1 Requesting a Puzzle](#request-puzzle)
        * [2.5.2 Submitting your Solution](#puzzle-submission)
        * [2.5.3 The Slow Solver](#puzzle-slow-solver)
* [3 Interrupts](#interrupts)
    * [3.1 Interrupt Acknowledgement](#interrupt-ack)
    * [3.2 Bonk](#bonk-ack)
    * [3.3 Request Puzzle](#request-puzzle-ack)
    * [3.4 Respawn](#respawn-ack)
* [4 SPIMBot Physics](#physics)
    * [4.1 Position and Velocity](#position-velocity)
    * [4.2 Collisions](#collisions)
* [5 Running and Testing Your Code](#running)
    * [5.1 Useful Command Line Arguments](#cl-args)
* [6 Tournament Rules](#tournament)
    * [6.1 Qualifying Round](#tournament-quals)
    * [6.2 Tournament Rounds](#tournament-rounds)
    * [6.3 LabSpimbot Grading](#grading)
* [Appendix A](#app-a)
* [Appendix B](#app-b)
* [Appendix C](#app-c)

1 The Game
==========

For this year's SPIMBot competition, you will be writing MIPS code for a bot to play a territorial game. You'll score points by maintaining ownership of more tiles than your opponent, and by knocking your opponent out by hitting them with bullets. Here are some fast facts:

* You'll play for **10 million** cycles
* Solving puzzles will earn you bullets
    * You can use them to take ownership of enemy tiles
    * You can also use them to shoot your opponent
* Shooting your opponent or getting shot doesn't stop the game
    * The field will reset, you'll get **17,000** points, and you'll keep playing

To help you think about how your bot should behave, here's an example of what a full game might look like. All the cycle numbers except for **0** and **10,000,000** are just examples, and the bots could shoot each other more or less depending on the match-up.

| Cycle | Event |
| --- | --- |
| 0   | The game begins. Bots each have **0** points, **0** bullets, and **50/50** tile ownership (see [**1.1 The Map**](#map)) |
| 1 - 73,432 | Bots compete for tile ownership and try to shoot each other. **Bot 1** earns some points for owning the most tiles on average (see [**1.5 Scoring Points**](#points)) |
| 73,433 | **Bot 0** shoots **bot 1** and earns **17,000** points. Everything except score resets to how it was in cycle 0 (see [**1.6 Resetting**](#reset)) |
| 73,434 - 608,404 | Bots continue to compete as before. This time, **bot 0** earns some points for owning the most tiles on average. |
| 608,405 | **Bot 0** shoots **bot 1** again and earns **17,000** points. Everything except score resets to how it was in cycle 0. |
| 608,406 - 10,000,000 | Bots continue to compete as before. This time, **bot 1** earns some points for owning the most tiles on average. |
| 10,000,000 | The game ends because **10 million** cycles have passed. **Bot 0** wins because it earned more points in total. |

Now that you know what a match might look like, let's dive into each of the components of the game. We recommend you read this entire manual so that you're prepared to write working code for your bot.

1.1 The Map
-----------

You'll see the map when you run QtSpimbot. It looks like this.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/maponly_gridded_SP22.png)

Here's what you should know:

* The map is a **40x40** grid of **tiles**
    * Each tile is made of **8x8 pixels**
* **Bot 0** (black, owns white tiles) always starts in the top left corner **(Tile \[3, 3\] = pixel \[28, 28\])**
* **Bot 1** (white, owns black tiles) starts in the bottom right corner **(Tile \[36, 36\] = pixel \[292, 292\])**
* Each player owns half the tiles to start
    * They can only walk on their own tiles
    * They can shoot bullets to flip tiles to their ownership or knock out their opponent

**Note:** During lab 9, grading, and tournament qualification, **your SPIMBot will always be assigned to be bot #0.** This does **NOT** apply for the tournament, though, so **don't forget** to make your bot take into account whether it's bot 0 or bot 1.

You'll also notice that some parts of the map don't always look the same between runs. These parts of the map are randomized **once at the start of the game** as shown in the image below. Areas of the map where randomized obstacles can spawn are painted red, while areas that are guaranteed to be made up of floor tiles are painted green. Once the game begins, the **obstacles will not change**.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/markedupmap_SP22.png)

Each of the red 4 by 4 tile regions will be populated with one of the following patterns. White grid squares represent walkable tiles, while blue squares represent non-walkable areas.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/randomtile_Sp22.png)

See [**1.3 Tile Types**](#tiles) below for a summary of tile types on the map.

1.2 The HUD
-----------

The HUD is displayed directly below the map and contains information about each player. The image below labels each component.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/hud_annotated.png)

Here's what each field means:

* **Score:** a player's score, increased by owning more tiles than the opponent, or hitting the opponent with bullets
    * Whoever has the higher score at the end of the game wins!
* **Reserve bullets:** the number of bullets a player has available to shoot
    * More are earned by solving puzzles
* **Active bullets:** the number of bullets a player has on the screen
* **Tile delta (ΔT):** the difference between a player's current tile ownership and their opponent's
    * every 100,000 cycles, **if ΔT > 0, score += ΔT** updates your score

1.3 Tile Types
--------------

Below is a summary of the different tile types:

| Tile type | Notes |
| --- | --- |
| Floor 0 | **Bot 0**'s tile, painted **white** on the map. **Bot 0 can walk** on these. Acts as a **wall for bot 1**. |
| Floor 1 | **Bot 1**'s tile, painted **black** on the map. **Bot 1 can walk** on these. Acts as a **wall for bot 0**. |
| Wall | An immovable wall, painted **blue** on the map. **Nobody can walk** here, and bullets cannot pass through. |

In short, each bot can only walk on tiles that are the **opposite** color of the bot, and nobody can walk through blue walls. A bot can take ownership of an opponent's tile by shooting it, and bullets won't stop until they hit a wall or the opponent. See **SHOOT** and **CHARGE_SHOT** in [**Appendix A**](#app-a).

You can access the map using the **GET_MAP** command. It'll return a struct that contains a 2D array with each entry corresponding to a tile on the map. The value of each entry will contain information about the corresponding tile, including the tile type. See [**Appendix A**](#app_a) and [**Appendix B**](#app-b) for more information!

1.4 The Objective
-----------------

The player with the most points after 10 million cycles of gameplay wins. **Your objective is to be that player.**

1.5 Scoring Points
------------------

You can earn points under two conditions:

1.  You **shoot** your opponent
    * You will earn **17,000 points**
    * Your opponent will not lose any points
    * Tile ownership will reset to how it was when you launched QtSpimbot (see [**1.6 Resetting**](#reset))
    * You will both be placed in your original starting corners
    * Scores will NOT be reset and you will play again repeatedly until 10,000,000 cycles pass
2.  You **own more tiles** than your opponent
    * This condition is checked when **(cycle % 100'000 == 100'000 - 1)**
    * If your ΔT (tile delta) is positive, **your score is incremented by ΔT**
    * Points will **not be lost** for having a negative ΔT

1.6 Resetting
-------------

When **either player** is **hit with a bullet**, the **map will reset** so that spawn camping (waiting near your opponent's starting point and shooting them immediately when they appear) is impossible.

When the map resets, the following things happen:

1.  Bots will be reset to their original state, but your MIPS code will continue running
    * Bots will be placed back in their starting positions **(tile \[3, 3\] and \[36, 36\])**
    * Bots will have their velocity reset to 0
    * Bots will have their angle reset to 90 (see [**4.1 Position and Velocity**](#physics))
    * Bots will have their reserve bullets reset to 0
    * Any progress towards a charged shot is reset (see [**1.7 The SPIMBot**](#bots))
2.  The **RESPAWN** interrupt will be raised for both bots (see [**Part 3, Interrupts**](#interrupts))
3.  Tile ownership will reset to its original state (see [**1.1 The Map**](#map))
    * Obstacles will **not** be re-randomized
4.  Gameplay will continue as before, and points will be kept.

1.7 The SPIMBot
---------------

The SPIMbot **executes MIPS code** you write for it and uses memory-mapped IO (MMIO) to interact with the world. This section is a summary of your bot's abilities; for detailed information on **MMIO commands**, see [**Part 2, SPIMBot MMIO**](#mmio).

Bots can move around within the tiles they own (Bot 0 owns white tiles, Bot 1 owns black tiles), but they cannot move outside these tiles. They can also use MMIO sensors to see where they are on the map and to check who owns which tiles.

In addition to moving around, bots can **solve puzzles**, which will grant them **bullets** (see [**2.5 Puzzle**](#puzzle)). They can then **launch bullets** in any of the four **cardinal directions** using the **SHOOT** MMIO. Doing so will remove **1** bullet from the bot's ammo reserve. These bullets can be used strategically to flip the enemy's tiles, increasing the area on which the shooter can move around (and decreasing it for their enemy). Bullets won't stop until they hit something, so they can capture multiple enemy tiles in a row. If two bullets fired by different players collide, the bullets will both be destroyed, and the same will happen if a bullet hits a wall.

Bots can also fire a **charged shot** of three bullets side-by-side using the **CHARGE_SHOT** MMIO, causing bullets to spawn at the bot's location as well as on the tiles directly beside them on either side. The bullets travel in the same direction, as shown below.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/chargedshot.png)

Doing this does not only consume **1** bullet but also requires the bot to wait **10,000** cycles between starting to charge their attack and releasing it. The process of firing a charged attack is as follows:

1.  Use the **CHARGE_SHOT** MMIO to start charging
    * You commit to which direction you'll fire the bullets here!
2.  Wait **10,000 cycles**; don't use **SHOOT** yet!
3.  Use **SHOOT** to release the charge

Take note that using **SHOOT** before 10,000 cycles have passed will just fire a single bullet and reset your progress towards a charged shot. Also, it doesn't matter what direction you use **SHOOT** with; it'll be ignored and the direction you committed to in step **1** will be used instead.

Bots can also read from **MMIO_STATUS** to see if the last command they used worked successfully. A value of **0** indicates success, and information on other possible values can be found in the starter MIPS code.

Now that you know roughly what your bot can do, let's take a closer look at all the available MMIO commands.

2 SPIMBot MMIO
==============

SPIMBot's sensors and controls are manipulated via memory-mapped IO; that is, the IO devices are queried and controlled by reading and writing particular memory locations. All of SPIMBot's devices are mapped in the memory range **0xffff0000 - 0xffffffff**. This section will describe the available MMIO devices; for a summary of all MMIO addresses, see [**Appendix A**](#app-a).

2.1 Orientation Control
-----------------------

SPIMBot's orientation can be controlled in two ways:

1.  By specifying an adjustment relative to the current orientation (**turn by** x degrees)
2.  By specifying an absolute orientation (**point at** x degrees)

In both cases, an integer value (between **-360** and **360**) is written to **ANGLE** and then a command value is written to **ANGLE_CONTROL**. If the command value is **0**, the orientation value is interpreted as a **relative** angle (the bot will **turn** by that amount). If the command value is 1, the orientation value is interpreted as an **absolute** angle (the bot will **point in** that direction). Note that writing to **ANGLE** will **not** do anything on its own; you need to write to **ANGLE_CONTROL** to actually make the bot turn.

Angles are measured in degrees, with **0** defined as facing right. **Positive angles** turn the bot **clockwise**. While it may not sound intuitive, this matches the normal Cartesian coordinates (the `+x` direction is 0°, `+y` is 90°, `-x` is 180°, and `-y` is 270°), since we consider the top-left corner to be (0,0) with `+x` and `+y` being right and down, respectively. For more details see [**Part 4, SPIMBot Physics**](#physics).

2.2 Odometry
------------

Your bot has sensors that tell you its **current position**. Reading from addresses **BOT_X** and **BOT_Y** will return your bot's x-coordinate and y-coordinate respectively, **in pixels** (not tiles!). Storing to these addresses, unfortunately, does nothing - no teleporting :)

2.3 Bonk (Wall Collision) Sensor
--------------------------------

The bonk sensor **raises an interrupt** whenever your bot **runs into a wall**.

**Note:** Your bot's velocity is set to zero when it hits a wall, so you'll need to get it moving again.

For information on interrupt masks and acknowledge addresses, see [this table](#interrupt-ack).

2.4 Timer
---------

You can do two things with the timer:

1.  Check the number of cycles since the start of the game by reading from **TIMER**.
2.  Request a **timer interrupt** (like setting an alarm) by writing the time you want to be interrupted to **TIMER**. This is great for switching between solving puzzles and moving around the map.

For information on interrupt masks and acknowledge addresses, see [this table](#interrupt-ack).

2.5 Puzzle
----------

Solving puzzles is the only way to get more **bullets**, which you may have noticed are essential for scoring points (see [**1.5 Scoring Points**](#points)). Each correct solution results in **1** bullet. You will be solving the Sudoku puzzle (The one in lab7). It's a similar task to Lab 7, but we have modified the puzzle struct slightly. **Your Lab 7 solver will not work without modification for specific edge cases! Use the provided solver instead!**

### 2.5.1 Requesting a Puzzle

The first step to getting a puzzle is to allocate space for the board in the data segment, then write a pointer of that space to **REQUEST_PUZZLE**. However, it takes some time to generate a puzzle, so you will have to wait for a **REQUEST_PUZZLE** interrupt. When you get the **REQUEST_PUZZLE** interrupt, the board for you to solve will be in the allocated space with the address you gave. Note that you must enable the **REQUEST_PUZZLE** interrupt or else you will never receive a puzzle.

To accept puzzle interrupts, you must turn on the mask bit specified by **REQUEST\_PUZZLE\_INT_MASK**. You must acknowledge the interrupt by writing any value to **REQUEST\_PUZZLE\_ACK**. The puzzle will then be stored in the pointer written to **REQUEST_PUZZLE**.

You can request more puzzles before solving the previous ones. But be sure to submit the solution in the same order as you requested them.

For information on interrupt masks and acknowledge addresses, see [this table](#interrupt-ack).

### 2.5.2 Submitting your Solution

After solving the puzzle, you need to submit the solution to obtain more water. To submit your solution, simply write the pointer to your board to **SUBMIT_SOLUTION**. If your solution is correct, you will be rewarded with a certain amount of bullets. To check if you were granted water for your solution, read from **MMIO_STATUS**. If reading from this address yields 0, your solution was correct. If it yields anything else, your solution was not correct.

Request a puzzle, and submit something to see examples of potential solutions.

For information on interrupt masks and acknowledge addresses, see [this table](#interrupt-ack).

### 2.5.3 The Slow Solver

We've given you a slow solver that you can use in your Spimbot. You can find it in **part2.s** under Lab 9 Part 2's **Lab Files To Download** section, along with the starter code. For all intents and purposes, it is just a solution to Lab7 and can be greatly optimized to give your bot an edge over the competition.

The arguments are the same ones as the solver given in Lab 7, and the solver is defined as the label `quant_solve` in **part2.s**.

3 Interrupts
============

The MIPS interrupt controller resides as part of co-processor 0. The following co-processor 0 registers (which are described in detail in section A.7 of your book) are of potential interest:

| Name | Register | Explanation |
| --- | --- | --- |
| Status Register | $12 | This register contains the interrupt mask and interrupt enable bits. |
| Cause Register | $13 | This register contains the exception code field and pending interrupt bits. |
| Exception Program Counter (EPC) | $14 | This register holds the PC of the executing instruction when the exception/interrupt occurred. |

3.1 Interrupt Acknowledgement
-----------------------------

When handling an interrupt, it is important to notify the device that its interrupt has been handled, so that it can stop requesting the interrupt. This process is called "acknowledging" the interrupt. As is usually the case, interrupt acknowledgment in SPIMBot is done by writing any value to a memory-mapped I/O location.

In all cases, writing the acknowledgment addresses with any value will clear the relevant interrupt bit in the Cause register, which enables future interrupts to be detected.

| Name | Interrupt Mask | Acknowledge Address |
| --- | --- | --- |
| Timer | 0x8000 | 0xffff006c |
| Bonk (wall collision) | 0x1000 | 0xffff0060 |
| Request Puzzle | 0x0800 | 0xffff00d8 |
| Respawn | 0x2000 | 0xffff00f0 |

3.2 Bonk
--------

You will receive the **Bonk** interrupt if your SPIMBot runs into a wall. Your SPIMBot's velocity will also be set to zero if it runs into a wall.

3.3 Request Puzzle
------------------

You will receive the **Request Puzzle** interrupt once the requested puzzle has been written into the provided memory address.

3.4 Respawn
-----------

When a bot successfully kills the other bot, then a **Respawn** interrupt is sent to both bots. The interrupt occurs if and only if the world is reset and the bots are respawned back at their starting locations. Ammo and shot bullets are also reset. The only thing that is preserved after a respawn is the bots' scores.

4 SPIMBot Physics
=================

4.1 Position and Velocity
-------------------------

In the SPIM Universe, positions are given in pixels. Pixels start in the upper left at `(x=0,y=0)` and end in the bottom right at `(x=320,y=320)`. Just as with the Cartesian plane, the **x-coordinate increases** as you go to the **right**. However, unlike the Cartesian plane, the **y-coordinate increases** as you go **down**. This is the convention for computer graphics, so we're sticking with it.

An angle of 0° is parallel to the positive x-axis. As the angle moves clockwise, it increases. When the angle is parallel to the positive y-axis (pointing **down**), it's at 90°.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/graphics_coordinates.png)

When we describe the position of a SPIMBot, we refer to the coordinates of its center. The bot is just a circle with a radius of 3 pixels centered around a given point. It has a little stem pointing off in one direction to indicate which way the bot is facing.

Your bot's velocity is measured in units of pixels/10,000 cycles. This means that at maximum speed (±10), the bot moves at a speed of 1 pixel per 1000 cycles, or 1 tile (8 pixels) per 8000 cycles.

The SPIMBot has no inertia and does not quite obey the laws of real-world physics. This means that you can rotate your bot instantly by using the **ANGLE** and **ANGLE_CONTROL** commands, and it can accelerate to a given speed instantaneously using the **VELOCITY** command.

4.2 Collisions
--------------

If your position is about to go out-of-bounds (either less than 0 or greater than 320 on either axis) or cross into an impassible cell, your velocity will be set to zero and you will receive a **bonk interrupt**. Your bot will walk straight through the opponent, but it will get hit by any enemy bullet that it shares a tile with.

**Note:** Your position is the center of your SPIMBot! This means that your bot will partially overlap the wall before it “collides” with it. That's right, bots don't have real hitboxes when it comes to colliding with walls.

5 Running and Testing Your Code
===============================

To specify the MIPS file that will control your bot, use the `-bot0` or `-bot1` arguments followed by the file's name. Be sure to put any other flags before this flag.

If you use `-bot0`, your bot will play as **Bot 0**. If you use `-bot1`, your bot will play as **Bot 1**. Again, it is important that your code works for bot 0 **and** bot 1, because you will be randomly assigned one or the other during each round of the tournament.

Besides the map window, you'll see one that looks like the one below. The button to start the simulation is labelled.

![](/pl/course_instance/145219/instance_question/380912362/clientFilesCourse/spimbot/images/QtSpimLabelled.png)

Lots of useful debug information will be printed to the command line, telling you what's going on in the game. You can disable this with `-nodebug` ("no debug", not "node bug"). You can also use the `-drawcycles` flag to slow down the action and get a better look at what is going on; try a number around **1000** to slow the game down a little or **100** to slow it down a lot more.

In addition, QtSpimbot includes two arguments (`-maponly` and `-run`) to facilitate rapidly evaluating whether your program is robust under a variety of initial conditions (these options are most useful once your program is debugged).

During the tournament, we'll run with the following parameters: `-maponly` `-run` `-tournament` `-randommap` `-largemap` `-exit_when_done` `-nodebug`

Note: the **-tournament** flag will suppress MIPS error messages!

Is your QtSpimbot instance running slowly? Try selecting the "Data" tab instead of the "Text" one.

Are you on Linux and having theming issues? Try adding `-style breeze` to your command line arguments.

5.1 Useful Command Line Arguments
---------------------------------

| Argument | Description |
| --- | --- |
| `-bot0 <file1.s> <file2.s> ...` | Specifies the assembly file(s) to use |
| `-bot1 <file1.s> <file2.s> ...` | Specifies the assembly file(s) to use for a second SPIMBot |
| `-part1` | Run SPIMBot under Lab 9 part 1 conditions |
| `-part2` | Run SPIMBot under Lab 9 part 2 conditions |
| `-qual` | Run SPIMBot under qualification conditions |
| `-test` | Run SPIMBot starting with 65535 bullets. Useful for testing. |
| `-nodebug` | Disable debug information printed to the command line |
| `-limit` | Change the number of cycles the game runs for. Default is 10,000,000. Set to 0 for unlimited cycles |
| `-randommap` | Randomly generate a scenario map with the current time as the seed. Potentially affect bot start position, scenario-specific positions, and general randomness. Note that this overrides `-mapseed` |
| `-mapseed <seed>` | Randomly generate scenario map based on the given seed. The seed should be a non-negative integer. Potentially affects bot start position, scenario-specific positions, and general randomness. Note that this overrides `-randommap` |
| `-randompuzzle` | Randomly generate puzzles with the current time as the seed. Note that this overrides `-puzzleseed` |
| `-puzzleseed <seed>` | Randomly generate puzzles based on the given seed. The seed should be a non-negative integer. Note that this overrides `-randompuzzle` |
| `-drawcycles <num>` | Causes the map to be redrawn every num cycles. Default is 8192, and lower values slow execution down, allowing movement to be observed much better |
| `-largemap` | Draws a larger map (but runs a little slower) |
| `-smallmap` | Draws a smaller map (but runs a little faster) |
| `-maponly` | Doesn't pop up the QtSpim window. Most useful when combined with `-run` |
| `-run` | Immediately begins the execution of SPIMBot's program |
| `-tournament` | A command that disables the console, SPIM syscalls, and some other features of SPIM for the purpose of running a smooth tournament. Also forces the map and puzzle seeds to be random. This includes disabling errors, which can make debugging more difficult |
| `-prof_file <file>` | Specifies a file name to put gcov style execution counts for each statement. Make sure to stop the simulation before exiting, otherwise the file won't be generated |
| `-exit_when_done` | Automatically closes SPIMBot when the contest is over |
| `-quiet` | Suppress extraneous error messages and warnings |
| `--version` | Prints the version of the binary (note the double-dash!)  <br>**Latest version:** 1.1.2 (Apr 10, 2022) |

**Tip:** If you're trying to optimize your code, run with `-prof_file <file>` to dump execution counts to a file to figure out which areas of your code are being executed more frequently and could be optimized for more benefit!

Note that `-randommap` and `-mapseed` override one another, and that `-randompuzzle` and `-puzzleseed` override one another. The `-tournament` flag also overrides most other flags. The flag that is typed last will be the overriding flag.

6 Tournament Rules
==================

6.1 Qualifying Round
--------------------

In qualification, your bot will play against a simple enemy three times. To pass qualifications, you will need to have at least two games where your bot either shoots the enemy or wins outright (or both). We will use the following command to run each of the three games.

    QtSpimbot -bot0 spimbot.s -bot1 adversary.s -mapseed X -puzzleseed Y -qual
    

Where X and Y represent a set seed. If you want to test with random seeds, you can use:

    QtSpimbot -bot0 spimbot.s -bot1 adversary.s -randommap -randompuzzle -qual
    

For details on how a bot wins a game, see [**1.4 The Objective**](#objective) and [**1.5 Scoring Points**](#points).

6.2 Tournament Rounds
---------------------

Once you have qualified, You will then have to compete in a tournament against your classmates. In each round, your bot will be randomly paired with another bot. The bot that has the highest score at the end of the round will win. In the case of a tie, the winner will be selected at random. The tournament might be round-robin, double-elimination, etc., depending on the number of people qualified.

We will use the following command to run two different bots against each other:

    QtSpimbot -bot0 spimbotA.s -bot1 spimbotB.s -tournament
    

**NOTE:** In tournament play, your bot may either be bot 0 or bot 1 (it could spawn in either starting corner), so don't forget to account for both scenarios in your code!

6.3 LabSpimbot Grading
----------------------

See the **Lab Spimbot Handout** for a breakdown of how your grade will be calculated.

Appendix A
==========

Below is a table of MMIO commands and their memory addresses. You can read or write to these addresses to get information about the game and perform in-game actions.

| MMIO | Address | Acceptable Values | Read | Write |
| --- | --- | --- | --- | --- |
| **VELOCITY** | 0xffff0010 | -10 to 10 | Current velocity | Updates velocity of the SPIMBot |
| **ANGLE** | 0xffff0014 | -360 to 360 | Current orientation | Updates angle of SPIMBot when **ANGLE_CONTROL** is written |
| **ANGLE CONTROL** | 0xffff0018 | 0 (relative)  <br>1 (absolute) | N/A | Updates angle to last value written to **ANGLE** |
| **TIMER** | 0xffff001c | Anything | Number of elapsed cycles | Timer interrupt when elapsed cycles == write value |
| **TIMER_ACK** | 0xffff006c | Anything | N/A | Acknowledge timer interrupt |
| **BONK_ACK** | 0xffff0060 | Anything | N/A | Acknowledge bonk interrupt |
| **REQUEST\_PUZZLE\_ACK** | 0xffff00d8 | Anything | N/A | Acknowledge request puzzle interrupt |
| **RESPAWN_ACK** | 0xffff00f0 | Anything | N/A | Acknowledge respawn interrupt |
| **BOT_X** | 0xffff0020 | N/A | Current X-coordinate, px | N/A |
| **BOT_Y** | 0xffff0024 | N/A | Current Y-coordinate, px | N/A |
| **OTHER_X** | 0xffff00a0 | N/A | Opponent's X-coordinate, tiles | N/A |
| **OTHER_Y** | 0xffff00a4 | N/A | Opponent's Y-coordinate, tiles | N/A |
| **SCORES_REQUEST** | 0xffff1018 | Valid data address | N/A | M\[address\] = \[your score, opponent score\] |
| **REQUEST_PUZZLE** | 0xffff00d0 | Valid data address | N/A | M\[address\] = new puzzle; sends Request Puzzle interrupt when ready |
| **SUBMIT_SOLUTION** | 0xffff00d4 | Valid data address | N/A | Submits puzzle solution at M\[address\] |
| **GET_MAP** | 0xffff2008 | Valid data address | N/A | Writes current **Map** struct to given memory location. |
| **MMIO_STATUS** | 0xffff204c | N/A | Returns the status of previously used MMIO. See the debug output in your terminal for more info. | N/A |
| **OTHER_BULLETS** | 0xffff200c | Valid data address | N/A | Returns list of opponent's bullets at M\[address\] |
| **BOT_BULLETS** | 0xffff2010 | Valid data address | N/A | Returns list of your bullets at M\[address\] |
| **GET_AMMO** | 0xffff2014 | N/A | Returns the number of bullets that the bot has in reserve (no max) | N/A |
| **CHARGE_SHOT** | 0xffff2004 | Direction Enum | Returns Direction Enum representing the direction of the shot is being charged (-1 if there is no ongoing charge) | Charge a shot towards the passed direction |
| **SHOOT** | 0xffff2000 | Direction Enum | N/A | Fires a shot. If the bot calls **CHARGE_SHOT** prior, the direction passed into **SHOOT** is ignored and fires a shot (fully charged or not) in the direction passed to **CHARGE_SHOT** |

Appendix B: Struct and Enum Definitions
=======================================

Some MMIO commands, like **GET_MAP**, write a struct to an address you provide, while others read a struct from an address you provide. The contents of any aforementioned structs are shown below.

    
    class ListNode {
        public:
        int pos;
        ListNode* next;
        ListNode(int pos, ListNode* next = nullptr) : pos(pos), next(next) {}
    };
    
    #define GRID_SQUARED 16
    
    class Sudoku : public Puzzle
    {
    public:
        Sudoku(int id);
        ~Sudoku();
        void write_to_bot(int context, mem_addr address);
        bool check_solution(int context, mem_addr address);
        unsigned id();
    
    private:
        unsigned short unsolved_puzzle[GRID_SQUARED][GRID_SQUARED];
        unsigned puzzle_id;
    };
    
    #undef GRID_SQUARED
    
    enum class Tile : char {
        FLOOR_0 = 0,
        FLOOR_1 = 1,
        WALL = 2
    };
    
    constexpr size_t MAP_SIZE = 40;
    
    struct Map {
        std::array, MAP_SIZE> arr; // Using tile enums
    };
    
    struct Bullet {
        char x_tile;
        char y_tile;
        char speed;
        char direction;
    };
    
    constexpr size_t MAX_ACTIVE_BULLETS = 15;
    
    struct ActiveBullets {
        int length;
        std::array arr;
    };
    

Appendix C: Helpful Code
========================

You might find some of the following functions helpful in writing your SPIMbot.

    .data
    three:  .float  3.0
    five:   .float  5.0
    PI:     .float  3.141592
    F180:   .float  180.0
    .text
    # -----------------------------------------------------------------------
    # sb_arctan - computes the arctangent of y / x
    # $a0 - x
    # $a1 - y
    # returns the arctangent
    # -----------------------------------------------------------------------
    .globl sb_arctan
    sb_arctan:
        li      $v0, 0      # angle = 0;
        abs     $t0, $a0    # get absolute values
        abs     $t1, $a1
        ble     $t1, $t0, no_TURN_90      
        ## if (abs(y) > abs(x)) { rotate 90 degrees }
        move    $t0, $a1    # int temp = y;
        neg     $a1, $a0    # y = -x;      
        move    $a0, $t0    # x = temp;    
        li      $v0, 90     # angle = 90;  
    no_TURN_90:
        bgez    $a0, pos_x      # skip if (x >= 0)
        ## if (x < 0) 
        add     $v0, $v0, 180   # angle += 180;
    pos_x:
        mtc1    $a0, $f0
        mtc1    $a1, $f1
        cvt.s.w $f0, $f0        # convert from ints to floats
        cvt.s.w $f1, $f1
        div.s   $f0, $f1, $f0   # float v = (float) y / (float) x;
        mul.s   $f1, $f0, $f0   # v^^2
        mul.s   $f2, $f1, $f0   # v^^3
        l.s     $f3, three      # load 3.0
        div.s   $f3, $f2, $f3   # v^^3/3
        sub.s   $f6, $f0, $f3   # v - v^^3/3
        mul.s   $f4, $f1, $f2   # v^^5
        l.s     $f5, five       # load 5.0
        div.s   $f5, $f4, $f5   # v^^5/5
        add.s   $f6, $f6, $f5   # value = v - v^^3/3 + v^^5/5
        l.s     $f8, PI         # load PI
        div.s   $f6, $f6, $f8   # value / PI
        l.s     $f7, F180       # load 180.0
        mul.s   $f6, $f6, $f7   # 180.0 * value / PI
        cvt.w.s $f6, $f6        # convert "delta" back to integer
        mfc1    $t0, $f6
        add     $v0, $v0, $t0   # angle += delta
        jr      $ra
        
    # -----------------------------------------------------------------------
    # euclidean_dist - computes sqrt(x^2 + y^2)
    # $a0 - x
    # $a1 - y
    # returns the distance
    # -----------------------------------------------------------------------
    euclidean_dist:
        mul     $a0, $a0, $a0   # x^2
        mul     $a1, $a1, $a1   # y^2
        add     $v0, $a0, $a1   # x^2 + y^2
        mtc1    $v0, $f0
        cvt.s.w $f0, $f0        # float(x^2 + y^2)
        sqrt.s  $f0, $f0        # sqrt(x^2 + y^2)
        cvt.w.s $f0, $f0        # int(sqrt(...))
        mfc1    $v0, $f0
        jr      $ra
    # -----------------------------------------------------------------------
    # form_word - given 0xAA, 0xBB, 0xCC, 0xDD, return 0xAABBCCDD
    # $a0 - 0x******AA
    # $a1 - 0x******BB
    # $a2 - 0x******CC
    # $a3 = 0x******DD
    # returns 0xAABBCCDD
    # -----------------------------------------------------------------------
    form_word:
        and     $a0, $a0, 0xFF
        sll     $a0, $a0, 24  
        ori     $v0, $a0, 0     # $v0  = 0xAA000000
        and     $a1, $a1, 0xFF  
        sll     $a1, $a1, 16
        or      $v0, $v0, $a1   # $v0 |= 0x00BB0000
        and     $a2, $a2, 0xFF  
        sll     $a2, $a2, 8     
        or      $v0, $v0, $a2   # $v0 |= 0x0000CC00
        and     $a3, $a3, 0xFF
        or      $v0, $v0, $a3   # $v0 |= 0x000000DD
        jr      $ra             #      = 0xAABBCCDD
