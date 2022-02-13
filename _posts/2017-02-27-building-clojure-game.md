---
layout: post
title: "Building a Clojurescript game"
description: ""
excerpt: "Let us implement a board game in ClojureScript, a functional programming language transpiling to JS, from the game rules to graphics to AI."
categories: [functional-programming]
tags: [Functional-Programming, Clojure]
---

In the next posts, we will implement a small web game using ClojureScript and Reagent. Our ultimate goal will be to build an equivalent of the old Tribolo DOS game as a Web game, including:

* The graphics and the rendering of the game board in SVG
* The game mechanics and the implementation of the rules
* The implementation of an artificial intelligence
* The interconnection of these aspects with state management

You can have a look at [the working version of the game](/tribolo/index.html).

Through these posts, we will cover the basics of building a SVG game with ClojureScript, talk about ways to architecture such an application, and try to remove any remaining traces of bad memories associated to GUI programming.

This endeavor will require more than one post to go through:

* In today’s post, we will describe the game and discuss our target architecture.
* In our next post, we will do a proof of concept of the target architecture by trying it on a much simpler game, a Tic-Tac-Toe.
* Then the next posts will dive into the implementation of the Tribolo itself, diving into each part of the implementation, one by one.

## The rules of Tribolo

The sections below will do into details in the rules. They turn out to be pretty hard to explain in plain text. You can experiment them first hand by [trying the game](/tribolo/index.html)) while reading the rules.

### Basic rules

Three players (Blue, Red and Green) play one after the other on board made of 16 times 11 cells. Each player tries to convert the maximum number of cells to their own color.

A player can convert cells of another player by “jumping” over them: the player targets an empty cell that has a direct line of sight (vertical, horizontal or diagonal) with one cell that he already controls.

At each turn, a player must do a valid jump. If no valid jump are available, the player pass his/her turn and the game proceeds with the next player.

### Jumping

To be a valid jump, a line must only contain cells belonging to one single opponent. It means that Blue cannot convert a line if the line contains both Red and Green cells. In addition, the line cannot contain cells that are not owned by any player.

If multiple jumps are possible from a given empty cell, converting this empty cell triggers the conversion of the cells of each line jumped over. It means that Blue can convert at the same time a line containing exclusively Green cells and a line containing exclusively Red cells.

The game also features one special type of cells: walls. Walls cannot be converted and block line of sight, which means a player cannot jump over them.

### Start and end conditions

The game starts with each player owning 12 random cells. The game continues until no player can convert any more cells. The winner is the player owning the most cells at the end of the game.

Each game has a fixed number of 12 walls, randomly placed on the board at the start of the game. Their position will never evolve.

### Additinal requirements

Tribolo is a single player game. Blue is controlled by the only human player of the game, while Red and Green are controlled by an artificial intelligence. We will have to develop at least a rudimentary AI to play the game.

Because the board is quite big, identifying available jump might sometimes be challenging. We will therefore add a help feature, to reveal the available moves for the the human player, as the original game does.

Finally, we want our human player to be able to cancel his last move or restart the same game from the start. We will add these features as actions available in a menu.

## Reagent

The GUI rendering of the game will be based on [Reagent](https://reagent-project.github.io/). We will thus need to spend some time providing an overview of the library.

There are plenty of good tutorials on Reagent available online, so we will only go through a refresher. You are greatly encouraged to look further on your own.

### Overview

Reagent is a lightweight wrapper around the amazing React library for JavaScript. It handles the rendering of the HTML DOM and automatically deals with the refresh of the HTML DOM when the state of the application changes.

Reagent is not concerned with the update of the application state. Data flows from the application state to the view and to Reagent, and not the other way around.

Modifying the application state has to be handled through the use of callbacks or events, either manually or via another library (like re-frame). We will have to take this into account in our architecture.

### Rendering with Reagent

From a user point of view, Reagent works in a beautiful simple way. Reagent understands a Hiccup-like format: it can transform nested data structures (vector and maps) that resemble the structure of an HTML tree, and build a HTML DOM from it.

If we provide Reagent with functions that produce HTML-like data structures, Reagent will wrap them in **reaction**. Reactions allow to automatically re-compute a function upon modification of a **ratom** on which the function depends.

A **ratom** is the name the community gave to the atom-like mutable state provided by Reagent. Each time a ratom is modified, rendering functions that dereference it will be automatically refreshed by Reagent.

### One single state to rule them all

The usual design with Reagent consists in having a single ratom state for the whole Single Page Application. This design works great in combination with [figwheel](https://github.com/bhauman/lein-figwheel), the must have lein plugging to hot reload code while coding.

If you are not accustomed to Reagent yet, this one single state design might seem a bit awkward or even plain bad to you. It is actually quite reasonable if you look at it from a data-base perspective:

* The single ratom represents the data-base for the application
* Reactions represents views on the data-model (projections, unions, etc.)
* The application queries these data-base views to get useful data
* The application runs transactions (swap!) on the DB to update the data

Having a single state also helps dealing with time. It is easier to deal with coordination when there is no need for coordination. It might not be the best fit for all applications, but it will suit for our Tribolo game.

## High level architecture

Now that we know what we have to build, and know more about Reagent, we can discuss the overall architecture of the game we will build.

The purpose of this architecture exercise is not to go deep into the details and set things into stones. Instead, it is to define “code zones”, along with the relations and the interaction patterns between these zones.

We define these zones to help us divide our tasks in small bits (problem decomposition) and also to enforce structurally some constraints that will help us better reason about and test our program.

### Identifying responsibilities

Our game will have to deal with several different responsibilities. We list them below with their corresponding short description:

* **Rendering**: transforming the current game state into HTML and SVG
* **Game logic**: the rules governing transitions between game states
* **AI logic**: selection of the best next move for an AI player
* **Store**: the in-memory data-base holding the application state
* **Game loop**: coordination and control of the different aspects

Each of these responsibilities will require further design refinement, which we will go through when the time comes to dive into each of them.

### Responsibilities dependencies

In order to keep the game under control both in terms of design and testability, we will impose ourselves some reasonable constraints in terms of design:

* The game logic is not allowed any dependency (on rendering, AI or store)
* The AI logic might depend on the game logic but nothing else
* The rendering should not depend the game loop
* Assembling the elements (dependency) is the job of the game loop

Given the set of constraints we listed above, we can select our target architecture as something very close to the usual model-view-controler:

* The game loop plugs the rendering with the ratom of the store
* The game loop listens the GUI and triggers game logic and AIs
* The game loop transmits the effect of game transitions to the store

This translates into the following ASCII Art diagram of interaction:

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

## The plan for the next posts

In this post, we described the rules of the game and covered the overall architecture we will follow to realize it.

Because it is dangerous to jump onto an architecture without second thoughts, our next post will go through a proof of concept of the architecture on a smaller game. We will implement a simple tic-tac-toe using it, to explain the architecture in more details, and see how it works.

The next posts will then dive deep into each of the responsibilities, one by one, covering them into enough details to reproduce them, until the game is done.
