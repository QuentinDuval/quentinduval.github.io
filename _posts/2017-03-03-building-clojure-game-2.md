---
layout: post
title: "Building a Clojurescript game (POC)"
description: ""
excerpt: "Let us implement a board game in ClojureScript, a functional programming language transpiling to JS, from the game rules to graphics to AI (part 2)."
categories: [functional-programming]
tags: [Functional-Programming, Clojure]
---

This post is the second in the series of posts focused on the conception and implementation of a port in ClojureScript of the game named Tribolo.

In our [previous post]({% link _posts/2017-02-27-building-clojure-game-1.md %}), we discussed the game, described its rule, and discussed its basic target architecture. This post will use this target architecture to build a much simpler game, a Tic-Tac-Toe, with the goal to make the architecture more explicit.

The Tribolo itself will indeed require several dedicated posts and so the architecture will be distilled through several posts. Building a Tic-Tac-Toe allows us to get the big picture through a Proof-Of-Concept.

To make our POC as useful as possible, our Tic Tac Toe will follow the same basic game mechanics and have the same kind of look and feel that our Tribolo game.

You can try both and see the parallel by yourself:

* You can try the Tribolo game we want to build at the [following address](/tribolo/index.html).
* You can try the Tic Tac Toe we will build today at the [following address](/tictactoe/index.html).

## Summary of the target architecture

We begin with a quick review of the target architecture we described in our last post. We first identified the following responsibilities:

* **Rendering**: transforming the current game state into HTML and SVG
* **Game logic**: the rules governing transitions between game states
* **AI logic**: selection of the best next move for an AI player
* **Store**: the in-memory data-base holding the application state
* **Game loop**: coordinates and control of the different aspects

Our target architecture uses the traditional decoupling between the view, the game logic (the model) and plugs the elements together though the game loop (the controler):

```
   Store ----------------> Rendering ---------------> Reagent
     ^      (reaction)         ^      (send hiccup)
     |                         |
     |                         | (listen)
     |       (update)          |
     \-------------------- Game loop
                               |
                               |
     ,-------------------------| (use to update store)
     |                         |
     v                         v
 Game logic <-------------- AI logic
```

This is it for the refresher. We will now instantiate this very simple architecture by implementing our Tic Tac Toe, to get a better feel of how it looks like in code.

## Tic Tac Toe

Tic Tac Toe is a pretty simple game. The interesting thing about Tic Tac Toe is that it is quite close to Tribolo. Both are turn based games, both feature a board, both have rules tied to the ownership of the cells of the board by the player, and both our games will feature an undo feature.

All of this makes the Tic Tac Toe a pretty good Proof-Of-Concept for our Tribolo game and its architecture. The only noticeable difference will be the absence of Artificial Intelligence in our Tic Tac Toe.

This part is left out for several reasons:

* It will save us time and is not needed to explain the architecture
* The AI is so simple that it would not inform much on the Tribolo AI

## Selecting data structures

We will first begin by designing our data structure. The goal is to model our problem, identifying the entities and the relations between them, to help us reason about the code better during implementation.

### Entities and relationships

We identify the following concepts: a cell, a board and a player.

A **cell** is something that a **player** can own and is identified by a coordinate (two integers). The game is made of 9 cells, which together form the **board**.

We are interested in the following relationships between these entities:

* The notion of ownership of a cell by a player
* Winning cell sets: a player owning one entire set wins the game

A **turn** represents a given state of Tic Tac Toe: the ownership of the cells by the different players, and the next player to play. We will define a **game** as a succession of **turn** with valid transitions between them.

### Possible representations

We can represent **cells** by their natural identifier on the board, their coordinate x and y. We can use the two keywords `:cross` and `:circle` to represent each of our **player**. Winning cell sets can be represented as sets of coordinates. A game can be represented as a vector of turn.

Then, there are at least two ways we can represent the ownership of cells by players:

*Choice 1*: We could choose to maintain two sets of cells, one for each player. A cell (x,y) being owned by the player P would be represented as (x,y) being in the set of cells associated to player P. Identifying a winner is done by checking if the set of a player includes a winning cell set.

Technically, a turn would be represented as:

```clj
{:board {:owner/cross #{[0 0] [1 0]} ;; Cells owned by "cross"
         :owner/circle #{[0 2]}}     ;; Cells owned by "circle"
 :player :owner/circle}              ;; Next player to play
```

*Choice 2*: We could choose to model the ownership of the cells the other way around. Each cell would be associated a owner, or `:none` if not owned. The board could be an associative container from cells to players that own them. Identifying a winner is done by accessing the map for each winning cell set, and check if it is fully owned by a single player.

Technically, a turn would be represented as:

```clj
{:board {[0 1] :owner/none,
         [1 2] :owner/none,
         [0 0] :owner/cross,
         [2 2] :owner/none,
         [0 2] :owner/circle,
         [1 1] :owner/none,
         [2 1] :owner/none,
         [1 0] :owner/cross,
         [2 0] :owner/none},
 :player :owner/circle}
```

There are potential variations on this representation as well. We could choose to omit the cells that do not have owners and use nil to represent the absence of owner. Or we could choose to use a vector of vector to hold the owners.

### Choice of board representation

For our implementation, we will follow *choice 2* and maintain a board associating cells to their respective owners. A game being is succession of turns. So our game will be a vector of maps that each represent a turn:

```clj
[{:board {[0 1] :owner/none,
          [1 2] :owner/none,
          ;; More associations ...
          [2 0] :owner/none},
  :player :owner/cross},
 {:board {[0 1] :owner/none,
          [0 0] :owner/cross,
          ;; More associations ...
          [2 0] :owner/none},
  :player :owner/circle}]
```

_Note: We could have tried the representation described in choice 1 or decided not omit cells that are not owned as well. If we tried these representations, we would need to provide a function to list explicitly all the valid coordinates of the board (for the rendering especially). But being more explicit about the valid coordinates allows to have a more generic implementation. For example, we could implement a more exotic Tic Tac Toe where the grid is replaced by a more funky shape. We encourage you to try this option._

## Rendering

Because ClojureScript, Reagent and Fighwheel offer together such a dynamic experience, we will start with the rendering of turn. Doing so will help us have quick feedback on the game logic as we develop it.

### Callbacks

A sound choice of architecture dictates us that our view must be independent of the statement management of the game, which we named **game store**.

The usual solution is to use some form of callbacks to decouple these two parts. Although we could make use of a protocol there, we will use a simpler alternative: a map holding the different callbacks functions.

In the rendering code that follows, this map of function will be referred to as callbacks. It will hold the keywords `:on-move`, `:on-restart`, `:on-undo` to represent each of the actions that can be triggered by the user.

### Main frame

We want the main frame of our game to be composed of a top panel, holding actions like restart game and undo last move, on top of the representation of the game board, a 3 times 3 grid.

We can represent this easily by creating a namespace frame holding a function render that simply delegates the rendering of the different parts to their associated namespaces, panel and board.

```clj
(ns tictactoe.view.frame
  (:require
    [tictactoe.view.board :as board]
    [tictactoe.view.panel :as panel]))


(defn render
  "Rendering the main frame of the game,
   takes as input the callbacks to trigger events"
  [{:keys [board] :as turn} callbacks]
  [:div
   [panel/render-top-panel turn callbacks]
   [board/render-board board callbacks]
   ])
```

### Top panel

Rendering the top panel of the game is pretty straightforward. We will display two buttons, surrounding a “Tic Tac Toe” title that changes to “Draw Game”, “Cross wins” or “Circle wins” when we reach the end of the game.

```clj
(defn- make-button
  [on-click txt]
  [:button.top-button {:on-click on-click} txt])

(defn render-top-panel
  "Render the top panel:
   * The restart game button
   * The title of the game
   * The undo button"
  [turn {:keys [on-restart on-undo]}]
  [:div.scores
   [make-button on-restart utils/circle-arrow]
   [:h1#title (title/get-title turn)]
   [make-button on-undo utils/back-arrow]
   ])
```

This code makes use of two callbacks `on-restart-event` and `on-undo-event` that are linked do their corresponding button. The caller of the frame will have to provide the associated callbacks in the map callbacks.

This code also makes use of utilities to create a circle arrow or back arrow. This code is not particularly interesting but is included [in this Gist](https://gist.github.com/deque-blog/93924f49165f6137f2a8e3517ad2a4f5) for completeness. The same goes for get-title that computes the title of the game available [in this Gist](https://gist.github.com/deque-blog/3428159e9e958f96c1a6e0b6b5774cc8).

### Board

Rendering the board of the game consists in creating a SVG panel and filling it with the SVG representation of each cells. Using some helper functions, we can write it in a pretty simple way:

```clj
(defn render-board
  "Render the board:
   * Creates a SVG panel
   * Render the cells in it"
  [board {:keys [on-move]}]
  (utils/square-svg-panel
    {:model-size board/size
     :pixel-size cst/board-pixel-size}
    (for [cell board]
      [cell/render-cell cell on-move]
      )))
```

This code makes use of the rendering functions for the cells. Each cell is rendered differently based on the owner of the cell:

```clj
(defn render-cell
  "Dispatch the rendering of the cell based on the player"
  [[coord owner :as cell] on-move]
  (let [renderer (case owner
                   :owner/cross render-cross
                   :owner/circle render-circle
                   render-square)]
    (renderer coord {:on-click #(on-move coord)})))
```

Although we could have used multi-methods to dispatch to the right rendering function, we choose not to. The number of player is fixed and we did not need any customization points.

To finish the rendering of the board, we need to take care of the cells. As this is a bit tedious to play with SVG, we will skip this part. You can refer to the link to the GitHub repo at the end of the post if you are curious.

## Game logic

We now have everything we need to display our game. It is great time to move to the implementation of the game logic.

We will build this game logic from the bottom up:

* Start with the board: the association of coordinates to their owner
* Continue with the turn: the core of the game logic and its rules
* Finish with the game: the succession of turn that make up a full game

### Board

Our implementation of the board will rely on some useful constants:

* The size of the board (width and height are equal)
* The vector of valid coordinates on the board
* The empty board where each cell has no associated owner

```clj
(def size "The size of the board" 3)

(def coordinates
  "All the valid coordinates on the board"
  (for [x (range size) y (range size)] [x y]))

(def coordinates? (set coordinates))

(def empty-board (zipmap coordinates (repeat :owner/none)))
```

The code of the board is rather boring. It merely consists in wrapping operations around the associative container from coordinates to owners. This wrapping is not necessary. We only introduced it to add some assertions and give more specific names.

```clj
(defn get-owner-at
  "Get the owner associated to the cell"
  [board coord]
  {:pre [(coordinates? coord)]}
  (get board coord))

(defn has-owner?
  "Check whether the coord has an owner associated to it"
  [board coord]
  {:pre [(coordinates? coord)]}
  (not= (get-owner-at board coord) :owner/none))

(defn convert-cell
  "Assign the cell [x y] to a new player"
  [board player coord]
  {:pre [(coordinates? coord)
         (not (has-owner? board coord))]}
  (assoc board coord player))

(defn full-board?
  "Verifies whether the board has any empty cell left"
  [board]
  (not-any? #{:owner/none} (vals board)))
```

### Turn

The turn represents the state of the game at a particular point. It holds both the state of the board and the next player to play. The main function attached to a turn is the ability to compute the `next-turn` from a turn and a coordinate.

The next turn is obtained by converting the cell targeted at the provided coordinate to assign it to the current player. This process only makes sense if the game is not over and if the target coordinate is valid (not owned yet and inside the board):

```clj
(defn next-turn
  "Convert a cell to the player color and switch player"
  [turn coord]
  (if-not (or (game-over? turn) (invalid-move? turn coord))
    (-> turn
      (update :board board/convert-cell (:player turn) coord)
      (update :player next-player))))
```

The function above makes use of several helper functions. We will focus on the most interesting one: `game-over?`. A game is over if either the board is entirely filled, or if a player has won:

```clj
(defn game-over?
  "The game is over if either:
   * The board is full
   * There is a winner"
  [{:keys [board]}]
  (or
    (board/full-board? board)
    (has-winner? board)))
```

Identifying the winner consists in verifying if a given player owns at least one winning combination of cells. This `get-winner` functin makes use of the `sole-owner` function to check if a combination of cells is entirely owned by one given player:

* We retrieve the owners of each of the cells
* We put these owners in a set to remove duplicates
* Singleton sets allow us to identify potential winners

```clj
(defn- sole-owner
  "Indicates whether all positions are owned by the same player"
  [board positions]
  (let [owners (set (map #(board/get-owner-at board %) positions))]
    (case owners
      #{:owner/circle} :owner/circle
      #{:owner/cross} :owner/cross
      nil)))

(defn get-winner
  "Return the winner, or nil if the game has none"
  [{:keys [board]}]
  (some #(sole-owner board %) winning-cell-sets))
```

The interesting part about this approach is that the rules specifying the winning conditions are entirely extracted as winning cell sets:

```clj
(def winning-diags
  [(filter #(= (first %) (second %)) board/coordinates)
   (filter #(= (dec board/size) (reduce + %)) board/coordinates)])

(def winning-rows (partition board/size board/coordinates))
(def winning-lines (algo/transpose winning-rows))
(def winning-cell-sets (concat winning-rows winning-lines winning-diags))
```

We can easily change these rules by playing with this data alone. For example we could decide that owning all the corners makes you a winner: to do so, we add a set of winning cell containing [0 0], [0 2], [2 0], and [2 2].

### Game

To finish modeling our Tic Tac Toe game logic, we are left with implementing the game, the succession of turns. This is quite straightforward:

* The start of the game is a vector with only the start turn (empty board)
* Playing a turn calls `next-turn` and adds a new value onto the stack
* The current turn of the game is therefore the one at the top of the stack
* To undo the last a move is to pop the last turn from the stack

```clj
(defn new-game []
  [turn/start-turn])

(defn current-turn
  [game]
  (peek game))

(defn play-move
  "Play current player move at the provided coordinate"
  [game coord]
  (if-let [new-turn (turn/next-turn (current-turn game) coord)]
    (conj game new-turn)
    game))

(defn undo-last-move
  "Remove the last game if there is enough game played"
  [game]
  (if (< 1 (count game)) (pop game) game))

(defn handle-event
  "Callback to dispath the event on the game"
  [game event]
  (cond
    (= event :restart) (new-game)
    (= event :undo) (undo-last-move game)
    :else (play-move game event)
    ))
```

## Plugging it together

Our game logic is done. The rendering is ready to display a turn in our game. We only need to deal with state management and plug all these different parts together.

The state management is pretty light. It consists of an `ratom` to hold the current state of the game, a `reation` to access the current turn of the game, and a few functions to deal with event handling:

```clj
(defonce app-state (reagent/atom (logic/new-game)))
(def current-turn (reaction (logic/current-turn @app-state)))
(defn send-event! [e] (swap! app-state logic/handle-event e))

(defn handle-event
  "Dispath the event to the game logic, yielding a new game"
  [game event]
  (cond
    (= event :restart) (new-game)
    (= event :undo) (undo-last-move game)
    :else (play-move game event)))
```

To finish up, plugging the GUI with the state management and the callbacks to the game logic is done in main rendering function:

```clj
(defn tic-tac-toe
  "Main entry point, assemble:
   * the game state
   * the game view"
  []
  [frame/render @store/current-turn
   {:on-restart #(store/send-event! :restart)
    :on-undo #(store/send-event! :undo)
    :on-move #(store/send-event! %)}])

(reagent/render [tic-tac-toe]
  (js/document.getElementById "app"))
```

We are done. Apart from some boring code that we did not listed in this post, the game is now fully functional (pun intended).

## Conclusion and what's next?

Through the implementation of a Tic-Tac-Toe, we went over the basic architecture we will follow to build our [Tribolo](/tribolo/index.html) game. You can find the full implementation of our Tic-Tac-Toe in this [GitHub repository](https://github.com/deque-blog/tictactoe/tree/master/src/cljs/tictactoe).

Doing this Proof-Of-Concept demonstrated us how simple and effective this architectural style is, and how good it is at separating concerns:

* Our game logic is pure and not concerned with state management
* Our GUI is only concerned about the turn representation
* The top namespace of the software is the only one knowing all the pieces

As far as the game logic is concerned, the events could come from the network
Now that our architecture is pretty clear and seems working, our next task will be to develop the Tribolo game based on it. This will be the subject of the next posts.
