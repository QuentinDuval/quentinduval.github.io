---
layout: post
title: "Building a Clojurescript Game Logic"
description: ""
excerpt: "Let us implement a board game in ClojureScript, a functional programming language transpiling to JS, from the game rules to graphics to AI (part 3)."
categories: [functional-programming]
tags: [Functional-Programming, Clojure]
---

This post is the third in the series of posts focused on the design and implementation of a port in ClojureScript of a board game named Tribolo.

In our [first post]({% link _posts/2017-02-27-building-clojure-game-1.md %}), we discussed the game, described its rules and came up with a basic target architecture. In our [second post]({% link _posts/2017-03-03-building-clojure-game-2.md %}), we tested this architecture on a Proof-Of-Concept. We built a Tic Tac Toe to make sure the architecture was sound.

This post will go through the development of the game logic of our Tribolo game, from the definition of the main entities and relationships, to the implementation choices made. The full game is available and playable at this address.

_Note: This post builds on top of the previous two posts. In particular, it assumes the reader understands the rules of Tribolo. You can refer to our first post to review these rules._

## Entities and relationships

Our first step before going into the implementation will be to think about the different concepts that make up the Tribolo game. Each of the words in **bold** below underlines an important concept of the game.

Like the Tic Tac Toe game we built in our earlier post, the game is all about **player** fighting for the ownership of **cells**. Cells can be identified by a two dimension coordinate. The **board** is the collection of the the cells of the game.

A **game** is made of a succession of **turn**. Each turn represents a valid point in time in a complete game. Unlike the Tic Tac Toe though, cells can change owners over the course of the game. The rules to transfer ownership of cells are specific enough to deserve a name.

We will designate as **transition** a valid progression from one turn a next turn. There might be several transitions available from a given turn. Each of these transition leads to a new turn. Playing a move can be viewed as following a transition.

A transition between turns consists in realizing one or several **jump**. Each jump does convert the set of cells it went over. A given jump can only convert cells belonging to one opponent.

The **score** of a player is the total amount of cells owned by the player. The player with the highest score at the final turn is the winner.

## High level design

In the previous section, we figured the concepts we want to represent. Let us try to refine them by associated them some kind of high level API. This API will be our guide during the implementation.

### Board

The Board allows to represent the concept of ownership of a cell. Much like in the Tic Tac Toe game, its main goal is to maintain a set of associations from coordinates to their associated owner. The main operations are:

* The creation of a new board
* The conversion of a cell to a new owner
* The retrieval of a cell’s associated owner
* The traversal of all associations (for rendering)

In Haskell-ish notation, it would match the following function prototypes (where IO characterizes impure functions):

```hs
new-board :: IO Board
convert-cell :: Board -> Coord -> Player -> Board
get-owner-at :: Board -> Coord -> Player
to-seq :: Board -> [(Coord, Player)]
```

### Turn

A turn represents a given state of the game. It gives access to the board, the scores, and the next player to play.

```hs
board :: Turn -> Board
player :: Turn -> Player
scores :: Turn -> Scores
```

The main operations on a turn are:

* To create a new starting turn (when we create a new game)
* To compute the transitions available from this turn
* To follow one of these transitions (playing a move)

```hs

new-init-turn :: IO Turn
transitions :: Turn -> Map Coord Transition
next-turn :: Turn -> Transition -> Turn
```

### Game

To finish up, we can provide a very simple API for the game. From a game, we should be able to extract the current turn, play a move at a given coordinate, and go back to the previous player move (undo).

```hs
new-game :: IO Game
current-turn :: Game -> Turn
play-at :: Game -> Coord -> Game
undo-player-move :: Game -> Game
```

## Board

We will start with the implementation of the board. The goal of this component is to maintain a list of associations from coordinates to their respective owners.

### Data structures

In the Tic Tac Toe, we choose to implement the Board with an associative container from coordinates to players. For the sake of experimenting something else, we will settle for a vector of vector of owners.

The outer vector stores the column by column index. The inner vectors stores the player by row index. The main advantages of using this representation is that we can give it a nice spec with the size included in it:

```clj
(def width 16)
(def height 11)

(s/def ::board
  (s/every
    (s/every ::player/owner :count height)
    :count width))
```

We can provide an instance of an empty board, which will be helpful when we will need to instantiate a new board:

```clj
(def empty-board
  "An empty board of `width` columns times `height` owners"
  (let [empty-column (vec (repeat height :none))]
    (vec (repeat width empty-column))))
```

### Specs of the API

Given the high level API we provided in the last section, we can derive the following specs for our board API:

```clj
(s/fdef new-board
  :ret ::board)

(s/fdef convert-cell
  :args (s/cat :board ::board
               :coord ::coord
               :owner ::player/player)
  :ret ::board)

(s/fdef get-owner-at
  :args (s/cat :board ::board
               :coord ::coord)
  :ret ::player/player)

(s/fdef to-seq
  :args (s/cat :board ::board)
  :ret (s/every (s/tuple ::coord ::player/owner)))
```

These specifications make use of the specification for coordinates (which makes use of the fact that sets are predicates in Clojure):

```clj
(def coordinates
  (vec (for [x (range width)
             y (range height)]
         [x y])))

(s/def ::coord (set coordinates))
```

### Implementation

Most of the implementation for the board API is straightforward. The only interesting bit is the implementation of the `new-board` function.

A new board must contain 12 randomly located walls, and 12 randomly chosen cells owned by each player. These four sets of 12 cells must be disjoint.

We implement a generic algorithm that will handle the task of picking n elements and attribute them to each element of groups. The core algorithm `pick-n-for-each` is free of randomness:

```clj
(defn zip
  "Create a sequence of tuples, each element being drawn from one collection"
  [& colls]
  (apply map vector colls))

(defn pick-n-of-each
  "Pick n `elements` for each of the `groups` starting from the beginning"
  [n elements groups]
  (zip elements (mapcat #(repeat n %) groups)))

(defn randomly-pick-n-of-each
  "Pick n `elements` for each of the `groups` randomly"
  [n elements groups]
  (pick-n-of-each n (shuffle elements) groups))
```

Because we isolated the randomness side-effect from the core algorithm, we can convince ourselves that it works properly by writing some tests around it:

```clj
(deftest test-pick-n-of-each
  (let [pick-n-of-each (comp vec algo/pick-n-of-each)]
    (testing "Varying number of each group"
      (are
        [expected n] (= expected (pick-n-of-each n (range) [:a :b]))
        [] 0
        [[0 :a] [1 :b]] 1
        [[0 :a] [1 :a] [2 :b] [3 :b]] 2
        ))
    (testing "Varying number of groups"
      (are
        [expected groups] (= expected (pick-n-of-each 2 (range) groups))
        [] []
        [[0 :a] [1 :a]] [:a]
        ))
    (testing "No elements to choose from"
      (is
        (= [] (pick-n-of-each 1 [] [:a]))
        ))))
```

We can then use the algorithm to implement our `new-board` function by choosing as elements the coordinates on the board, and as groups the players plus the wall:

```clj
(defn new-board
  "Creates a new board with initial positions of each players"
  []
  (->>
    (conj player/all :wall)
    (algo/randomly-pick-n-of-each cst/init-block-count coordinates)
    (reduce
      (fn [board [coord owner]] (convert-cell board coord owner))
      empty-board)))
```

## Turn

We will continue our implementation with the turn. The turn is the component that hosts most of the game logic of our Tribolo. It is quite large. So we will split its implementation in two:

* This section will deal with the turn itself and its representation
* The next section will zoom on the computation of the transitions

### Data structure

A turn gives us access to current state of the board, the next player to play and the scores. We will use a simple map to store that information, and encode its schema in the following spec:

```clj
;; Imported from "triboard.logic.scores.cljs"
(s/def ::scores
  (s/map-of ::player/player int?))

(s/def ::turn
  (s/keys :req-un
    [::board/board
     ::player/player
     ::scores/scores]))
```

The representation of transitions is a bit trickier. There are many different possible implementations available. We could represent transitions as functions from turn to turn. We could try to create a protocol for it.

But because Clojure loves data, we will represent these transitions as data. A transition will be a collection jumps, with each jump being a standard map, holding:

* The destination of the jump (an empty cell)
* The player winning the cells jumped over
* The player loosing the cells jumped over
* The coordinates of the cells jumped over

We can therefore describe a jump and a transition using the specs that follow:

```clj
(s/def ::destination ::board/coord)
(s/def ::taken (s/coll-of ::board/coord))
(s/def ::winner ::player/player)
(s/def ::looser ::player/player)

(s/def ::jump
  (s/keys :req-un [::destination ::taken ::winner ::looser]))

(s/def ::transition (s/every ::jump))
```

### Specs of the API

We can also spec the main functions of our Turn API, based on the previous specs for jump and transitions (which we isolated to the namespace transition):

```clj
(s/fdef next-turn
  :args (s/cat :turn ::turn
               :transition ::transition/transition)
  :ret ::turn)

(s/fdef transitions
  :args (s/cat :turn ::turn)
  :ret (s/map-of ::transition/destination ::transition/transition))
```

### Implementatoin (transition)

We might expect the transitions function to be quite complex. Its implementation is in fact quite simple: we only access the field of the turn data structure:

```clj
(defn transitions
  "Return the transitions available for the next player"
  [turn]
  (:transitions turn))
```

This is due to the rules of Tribolo. A player may not have any available transition to play. If so, he passes his turn and we consider the next player in the list. We must then repeat the same again or move to the next player.

As a consequence, to implement `next-turn `and find the next player to play, we must compute the transitions of the newly created turn. We keep this computation cached inside the turn for optimization purposes.

### Implementation (next-turn)

To compute the next turn of a game from a previous turn and a transition, we will go through three steps:

* We modify the board to take into account the jumps
* We modify the scores to be aligned with the new board
* We find the next player to play by looking at available transitions

```clj
(defn next-turn
  "Apply the transtion to the current turn, yielding a new turn"
  [turn transition]
  (-> turn
    (update :board transition/apply-transition transition)
    (update :scores scores/update-scores transition)
    (with-next-player)))
```

The functions `apply-transitions` and `update-scores` scan the transitions and update the board and the scores respectively. The interesting part is the computation of the next player, which we will describe below.

### Implementation (next-player)

As discussed earlier, finding the next player to play requires to evaluate the transitions available from our current turn.

We search for the first player that has at least a move to play. We temporarily assume the existence of a function `transition/all-transitions` that return a map from player to their respective available transitions.

```clj
(defn- who-can-play
  "Given the current player and the transitions for the next turn,
   returns the next player that has at least one transition available"
  [player transitions]
  (filter                         ;; Keep only the players which
    #(contains? transitions %)    ;; have at least a valid move to play
    (player/next-three player)))  ;; from the next three players

(defn- with-next-player
  "Complete the turn to compute to find the next player than can act
   * Requires to look-ahead for the next possible transitions
   * To save computation, keep this look-ahead available"
  [{:keys [board player] :as turn}]
  (let [transitions (transition/all-transitions board)
        next-player (first (who-can-play player transitions))]
    (merge turn
      {:transitions (get transitions next-player) ;; Keep the transitions of player
       :player next-player})))                    ;; Inside the turn (caching)
```

This ends up the implementation of the turn. The only missing part is the implementation of `all-transitions`. This is the subject of the next section.

## Computing transitions

We will now discuss the implementation of the Tribolo game logic core: the identification of all the valid transitions from a given board. This means identifying all the possible jumps that can be done by any player, before grouping them by winning player and jump destination coordinate.

### Performance constraints

The computation of the transitions from a given turn is the most CPU intensive task of the whole game. This will be especially true when we will go into the design and implementation of our AI.

As a consequence, the code that follows makes use of JavaScript arrays over ClojureScript vectors. This simple optimization trick brought nearly a 2X improvement on the AI. The same motivation is behind the use of low level loop-recur constructs in the following algorithms.

### Algorithm

The goal of the algorithm is to identify the jumps and then to group them by jump destination and player. A potential jump is a contiguous sequence of cells owned by the same player. It becomes a real jump if it is surrounded by:

* A cell with no owner: the destination of the jump
* A cell with another owner: the winner of the jump

This contiguous sequence of cell must follow an row, a column, or a diagonal of the board. A wall on either side means there can be no jump.

### Thoughts on implementation

We can visualize an implementation based on the use of the partition and partition-by algorithms:

* Using partition-by on cells to identify contiguous sequences of owners
* Using partition to group contiguous sequences by window of three

This idea is demonstrated below in the REPL on the first row of a sample board. It would have to be done for each row, each column, and each diagonal:

```clj
;; Partitioning by owner of the board gives contiguous sequences
(let [first-line (map vector (repeat 0) (range 0 16))]
  (partition-by #(board/get-owner-at board %) first-line))

;; Contiguous sequences of coordinates
=>
(([0 0]) ([0 1] [0 2] [0 3] [0 4]) ([0 5])
 ([0 6]) ([0 7]) ([0 8] [0 9]) ([0 10])
 ([0 11] [0 12] [0 13] [0 14] [0 15]))

;; Making some windows out of it
(partition 3 1 *1)
=>
((([0 0]) ([0 1] [0 2] [0 3] [0 4]) ([0 5])) ;; First window
 (([0 1] [0 2] [0 3] [0 4]) ([0 5]) ([0 6])) ;; Second window
 (([0 5]) ([0 6]) ([0 7]))
 (([0 6]) ([0 7]) ([0 8] [0 9]))
 (([0 7]) ([0 8] [0 9]) ([0 10]))
 (([0 8] [0 9]) ([0 10]) ([0 11] [0 12] [0 13] [0 14] [0 15])))
```

This algorithm would “touch” every cell 4 times (one for each direction), leading to 16 times 11 times 4 cell reads, for a total of 704. This algorithm is nice and fine, but it is not the one we will implement. I could not find a way to make it fast enough.

### Chosen algorithm

The algorithm we will implement is much more naive than the one described above. We will scan each empty cell and check in each direction if it is the destination of a jump.

This is not optimal: we will touch the cells more times than with our previous algorithm. It will require around 1200 cell reads at the start of the game, going down when the number of empty cells decreases.

This also looks quite naive as we could try to keep some of that computation between turns. It would require tracing what jumps are invalidated by another jump, which does not seem trivial.

### Scanning and grouping

We will start by the implementation of the outer loop of the algorithm. This loop will scan every single cell of the board, compute all the available jumps from this position, and group them by player and jump destination.

As indicated before in the performance constraints section, this algorithm needs to be pretty fast, and makes use of a conversion to a JavaScript array at the start:

```clj
(defn all-transitions
  [board]
  (let [aboard (board/board->array board)]    ;; Convert to JS array
    (transduce
      (mapcat #(available-jumps-at aboard %)) ;; Compute jumps at a given position
      (group-by-reducer :winner :destination) ;; Group by jump winner and destination
      board/coordinates)))
```

This code makes use of the `group-by-reducer` helper function, which allows to group elements by on several projections into a nested map as output:

```clj
(defn group-by-reducer
  "Create a reducer that groups by the provided key"
  [& key-fns]
  (fn
    ([] {})
    ([result] result)
    ([result val]
      (let [keys ((apply juxt key-fns) val)]
        (update-in result keys conj val)))))
```

### Finding jumps from a destination point

To find the jumps available at a given position, the destination of the jump, we start at an empty cell (not owned by any player).

We then move from the destination point in a fixed direction. The first player we meet is the looser of the potential jump. We continue until we find a cell with a different owner. This player is the winner of the jump.

We collect the coordinates along the way to identify the “jumped over” cells (in the taken vector) until we reach the second player. In case we hit a wall or another empty cell first, we stop the exploration: there is no valid jump in the searched direction. We also watch for going out of the board.

This algorithm translates to the following code, which searches for a jump in a fixed direction from the destination of the jump:

```clj
(defn- ^boolean block-jump? [owner]
  (or (= owner :none) (= owner :wall)))

(defn- ^boolean jump-start? [looser owner]
  (and looser (not= looser owner)))

(defn- ^boolean in-board? [x y]
  (and (< -1 x board/width) (< -1 y board/height)))

(defn- seek-jump-source-toward
  "Starting from the destination of a jump (a non-owned cell):
   * Search for valid source for the jump
   * Collect the jumped cells along the way"
  [board [x-init y-init :as jump-destination] [dx dy]]
  (loop [x (+ x-init dx)
         y (+ y-init dy)
         looser nil
         taken []]
    (if (in-board? x y)
      (let [owner (aget board x y)]
        (cond
          (block-jump? owner) nil
          (jump-start? looser owner) {:winner owner
                                      :looser looser
                                      :destination jump-destination
                                      :taken taken}
          :else (recur
                  (+ x dx)
                  (+ y dy)
                  owner
                  (conj taken [x y])))
        ))))
```

We have to repeat this algorithm for each possible jump direction. We ignore the empty cells since they cannot be the destination of a jump:

```clj
(defn- ^boolean empty-cell?
  [board [x y]]
  (= :none (aget board x y)))

(defn- available-jumps-at
  "Provides the list of jumps that can be done at a given `jump-destination`"
  [board jump-destination]
  (if (empty-cell? board jump-destination)
    (eduction
      (keep #(seek-jump-source-toward board jump-destination %))
      cst/directions)))
```

## Game

The last missing piece of the game logic is the implementation of the game. The game is simply a succession of turn. We can translate the high level API we gave to it into the following specs:

```clj
(s/def ::game (s/coll-of ::turn/turn))

(s/fdef current-turn
  :args (s/cat :game ::game)
  :ret ::turn/turn)

(s/fdef play-move
  :args (s/cat :game ::game
               :coord ::board/coord)
  :ret ::game)

(s/fdef undo-player-move
  :args (s/cat :game ::game)
  :ret ::game)
```

Its implementation is not terribly thrilling. It is like the implementation we used for Tic Tac Toe with one twist: the undo may go back several turns to find the last turn of the human player. The implementation is listed in this [GitHub Gist](https://gist.github.com/deque-blog/b23732d75a2a1ab4f06242202faa37b6) for completeness.

## Conclusion and what's next?

In this (pretty long) post, we went over the complete game logic of the Tribolo game, from the high level design to the implementation.

In the next posts, we will continue this journey and see how we can implement the other aspects of the game (Artificial Intelligence, User Interface, Game Loop, etc.).

But before we do that, we will spend the next post on discussing the use we made of Clojure Spec inside the game logic we implemented today.
