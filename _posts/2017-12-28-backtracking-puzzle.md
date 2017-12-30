---
layout: post
title: tiling and backtracking
feature-img: "assets/img/backtracking/tilingPuzzle1.jpg"
#thumbnail: "assets/img/thumbnails/tilingPuzzle1.jpg"
tags: [coding, fun, games]
---

Recently, I tried to solve one of those tricky tiling puzzles.
The task is to put 12 wooden pieces in a rectangular box such that the tiles do not overlap and fill out the box completely.
Each tile is made of 5 unit squares (actually, it consists of 5 unit cubes, but the third dimension doesn't play a role) and the size of the box is 10 x 6 (measured in units of a unit square).

The official solution to the puzzle is shown in this picture.

![official solution]({{ site.baseurl }}/assets/img/backtracking/tilingPuzzle2.jpg)

What I find surprising is that this solution has a simple symmetry: the rectangular box can be divided into two equally sized rectangles of size 5 x 6 such that the tiling is also a tiling for each individual sub-rectangle.
This is rather unnatural, because in addition to the boundary condition of the rectangular box the tiling must obey another boundary condition in the middle.
The shape of the tiles does not seem to be 'fine-tuned' for this additional constraint.
This made me wonder

* Is the solution unique (modulo obvious symmetry transformations)?
* Do all tilings obey the sub-rectangle symmetry?

It turns out that solving the puzzle by hand is not at all straightforward (at least not for me).
Since I could not come up with an idea to solve these questions in a clever way, I decided to write a python script that solves the tiling and helps to answer the questions.

Programming the puzzle is a standard exercise to learn backtracking.
You can find my (dirty) python script [on GitHubGist](https://gist.github.com/eniacization/e37b34d7a56957f727a21b8cbe4e2f5c).
Let's see how this works.

The output of

```python
board = Board(10, 6)
tiles = [cross,
         lShape,
         stick,
         tShape,
         mShape,
         hShape,
         zShape,
         uShape,
         angle,
         yShape,
         duck,
         boat]

solutionsBacktracking = []
solveBacktracking(board, tiles, solutionsBacktracking)
solutionsBacktracking[0].display()
```

is

```python
|  5 |  5 |  5 | 12 | 12 | 12 | 12 | 11 | 11 | 11 |
|  5 |  1 |  5 |  2 |  2 | 12 |  4 | 10 | 11 |  9 |
|  1 |  1 |  1 |  2 |  4 |  4 |  4 | 10 | 11 |  9 |
|  3 |  1 |  6 |  2 |  4 |  7 |  8 | 10 | 10 |  9 |
|  3 |  3 |  6 |  2 |  7 |  7 |  8 |  8 | 10 |  9 |
|  3 |  3 |  6 |  6 |  6 |  7 |  7 |  8 |  8 |  9 |
```

which is another valid solution that does not obey the sub-rectangle symmetry.

![new solution]({{ site.baseurl }}/assets/img/backtracking/tilingPuzzle3.jpg)

In the first two lines, we create a board of size 10 x 6 and fill a list with all 12 tiles.
Don't worry too much about the implementation of the board and the tiles, you can treat these objects as blackboxes.
If you are interested in the details you can read the [full script](https://gist.github.com/eniacization/e37b34d7a56957f727a21b8cbe4e2f5c).
Each tile object comes with a list of different shapes, e.g. `lShape.shapes` is a list of all shapes that can be obtained from the `lShape` tile by rotating and flipping the tile.

The heart of the program is the recursive function `solveBacktracking`.

```python
def solveBacktracking(board, tiles, solutions, maxSolutions = 1, times = None):
    if len(tiles) == 0:
        if Board.addSolution(solutions, copy.deepcopy(board)):
            print("New Solution! Solution number {}:".format(len(solutions)))
            if times is not None:
                times.append(time.time())
        if len(solutions) >= maxSolutions:
            return True
        return False

    for tile in tiles:
        for shape in tile.shapes:
            for pos in [(i, j) for i in range(board.width - len(shape[0]) + 1) for j in range(board.height - len(shape) + 1)]:
                if board.safeToPlace(shape, pos):
                    board.placeShape(shape, pos)
                    if board.noGaps() and board.lastShapeConnected():
                        if solveBacktracking(board, [t for t in tiles if t != tile], solutions, maxSolutions, times):
                            return True
                    board.removeLastShape()

    return False
```

The base case of the recursion is handled in the `if len(tiles) == 0` block.
That is, when no tiles are left the method `addSolution` checks whether the board is really a new solution and not already contained in the list `solutions`.
If the number of solutions reaches `maxSolutions` the algorithm terminates.

The recursion is handled within the loop over all remaining tiles.
For each remaining `tile`, each possible orientation `shape`, and each position `pos` of that shape inside the board the function `board.safeToPlace(shape, pos)` checks whether the shape can be put inside the box without overlapping other tiles already inside the box.
After the tile is placed via `board.placeShape(shape, pos)`, the line

```python
if board.noGaps() and board.lastShapeConnected():
```

checks whether the current tile configuration inside the box has no gaps (these gaps are dangerous as it might be impossible to fill them) and whether the last tile is connected to any of the previous tiles already inside the box.
If the two conditions are met, the recursion kicks in by using `solveBacktracking` to solve the simpler problem with one of the tiles removed.

If the algorithm runs into a dead end before finding the required number `maxSolutions` of solutions, the placement of the last tile is undone with `board.removeLastShape()` and the algorithm continues from this configuration.
Hence the name 'backtracking'.

You might wonder if the condition

```python
if board.noGaps() and board.lastShapeConnected():
```

is really necessary?
I don't thinks so.
But I find this condition very natural in the sense that it describes how I would play the puzzle.

A different version of `solveBacktracking` that does not check `board.noGaps()` and `board.lastShapeConnected()` is given by `solveBruteForce`.

```python
def solveBruteForce(board, tiles, solutions, maxSolutions = 1, times = None):
    if len(tiles) == 0:
        if Board.addSolution(solutions, copy.deepcopy(board)):
            print("New Solution! Solution number {}:".format(len(solutions)))
            if times is not None:
                times.append(time.time())
        if len(solutions) >= maxSolutions:
            return True
        return False

    tile = tiles[0]
    for shape in tile.shapes:
        for pos in [(i, j) for i in range(board.width - len(shape[0]) + 1) for j in range(board.height - len(shape) + 1)]:
            if board.safeToPlace(shape, pos):
                board.placeShape(shape, pos)
                if solveBruteForce(board, tiles[1:], solutions, maxSolutions, times):
                    return True
                board.removeLastShape()

    return False
```

The main difference between `solveBruteForce` and `solveBacktracking` is that the former tries all tiles in a fixed order, while the latter tries to build a growing connected island of tiles in the sea of an empty board.
One disadvantage of trying all tiles in a fixed order is that it is difficult to recognize and avoid nonsense configurations early in the game.
Trying all tiles in a fixed order is certainly not the way a human being would play the game.
On the contrary, building a connected island of tiles with no fixed order avoids some nonsense configurations but has the drawback of double counting.
Essentially, different orders in which the tiles are attached to the island can lead to identical configurations but are checked separately.

Which of the two, `solveBruteForce` or `solveBacktracking`, is faster?
It turns out that `solveBacktracking` finds the first solution to the puzzle in 13 minutes (on my old notebook), while `solveBruteForce` did not succeed to find any solution after more than one hour.

However, for the simpler puzzle

```
board = Board(5, 6)
tiles = [lShape, stick, zShape, uShape, yShape, boat]
```

`solveBruteForce` is much faster and finds all 24 solutions in just 6 seconds, while `solveBacktracking` needs 20 seconds to find these 24 solutions.

![Geometric pattern with fading gradient]({{ site.baseurl }}/assets/img/backtracking/BacktrackingVsBruteForce.png)

I suspect that `solveBruteForce` is typically faster if you want to find all solutions of a tiling problem, while `solveBacktracking` might be faster to find just one solution.
To faithfully answer the question, a careful analysis is needed.
One could for example investigate the average, best and worst case complexity depending on the tiling problem, the order in which the tiles are given, and the number of solutions.

There are certainly ways to increase the speed.
For example one could improve `solveBruteForce` by trying to recognize and discard nonsense configurations early in the game.
Or one could try to prune the double counting in the `solveBacktracking` algorithm by cashing configurations in order to recognizing identical tile configurations that differ only in the tile order.
If you have any ideas, why not share them in the comments?

Have fun.
