# nn-holdem
Code to build and teach a neural network to play a game of texas hold'em. The code includes a bare-bones hold'em table that even human players can play on via console.

As of this writing, it has all of the features it needs to run a proper single table game against a set of ai opponents.

### Current Status
The following things need to be built before the project is complete

* ~~hold'em dealer~~
* ~~pot splitting~~
* ~~incorporate hand rank evaluator~~ (using forked package [dueces](https://github.com/alexbeloi/deuces/tree/convert2to3) converted to python3)
* ~~neural network~~
* learning system

Additional (optional)

* learning from existing real world game history
* competition heuristics

### Usage
Running from play.py is simplest way to test things out (although the ai opponents are just random for now)
```python
$ cat play.py
from holdem import Table, TableProxy, PlayerControl, PlayerControlProxy

seats = 8
# start an table with 8 seats
t = Table(seats)
tp = TableProxy(t)

# controller for human meat bag
h = PlayerControl("localhost", 8001, 1, False)
hp = PlayerControlProxy(h)

print('starting ai players')
# fill the rest of the table with ai players
for i in range(2,seats+1):
    p = PlayerControl("localhost", 8000+i, i, True)
    pp = PlayerControlProxy(p)
```

To start an 8 person table with yourself + (seven) unlearned ai opponents simply run
```python
$ python3 play.py
Player  1  Joining game
starting ai players
Player  2  Joining game
Player  3  Joining game
Player  4  Joining game
Player  5  Joining game
Player  6  Joining game
Player  7  Joining game
Player  8  Joining game
Player 4 ['raise', 1475]
Player 5 ['fold', -1]
Player 6 ['fold', -1]
Player 7 ['fold', -1]
Player 8 ['fold', -1]
Stacks:
1 :  2000(P)(Button)(me)
2 :  1990(P)
3 :  1975(P)
4 :  525(P)
5 :  2000
6 :  2000
7 :  2000
8 :  2000
Community cards:  
Pot size:  1510
Pocket cards:   [ 3 ♠ ] , [ 3 ♣ ]  
To call:  1475
1) Raise
2) Call
3) Fold
Choose your option:
```

Currently designed to save the weight matrix (.npy) of the neural network if an ai opponent is the last left standing.

### Holdem Implementation

We built a basic single table no limit hold'em game. Both dealer and player run a SimpleXMLRPCServer to network board state and player moves.

The current implementation is meant to simulate a cash game. In the future, we will expand to accomodate multi-table tournament play.

## Neural network

The neural network uses mixed binary and continuous data.

Based on the recommendation of some literature on modeling systems with mixed data, we use *effect coding* **{-1,1}** instead of *dummy coding* **{0,1}** for the binary variables. For the continuous variables, we normalize by the bigblind and center all values. Ideally we would center values around their means, for the stack sizes we can use the mean stack size at any given table but for other continuous inputs are more tricky (currently trying to center around mean stack size anyway).

The activation function we're currently using is **tanh**, but since we aren't going to use backpropogation we may want to consider nondifferentiable activation functions.

### Input data

| Continuous      | Description |
| :---------------| :-----------|
| Pot             | Chips available to win in current hand of play |
| To Call         | Amout of chips needed to add to pot in order stay in current hand of play |
| Last Raise      | The most recent raise ammount for current round |
| Player Stacks   | Ammount of chips(money) each player has |
| BigBlind        | Size of minimum stake |

| Binary          | Description |
| :---------------| :-----------|
| Player position | Own position at the table |
| Pocket cards    | Cards in personal hand |
| Community cards | Shared cards available for all players to use |
| Button          | Position of player last to act in a round, determines the order of betting |

### Layers

We expect the number of nodes needed in each layer to be about **80-X-1**. The output will be interpreted as a bet amount, which will then be parsed as either raise, call, check, fold.

Alternative would be an **80-X-5** setup with the first four outputs in *[0,1]* denoting network confidence in raise, call, check, fold (resp.) and the last output denoting bet ammount normalized to personal stack or pot size.

### Learning

We plan to teach networks through coevolution + hall-of-fame. We spawn *~10000* random networks and have them compete with each other in 8-seat tables, we pick the top *100* networks and use them to spawn *10000* additional child agents through randomly biased averaging of the parent's weights and repeat with new generation of child agents + hall-of-famers.

The hall-of-fame agents are always preserved and never fall out of use. As the hall-of-fame becomes large, we randomly choose which hall-of-famers to include in the next generation, but the total pool of hall-of-famers is never allowed to diminish.

## Literature
* coevolution and hall-of-fame heuristics:
http://www2.cs.uregina.ca/~hilder/refereed_conference_proceedings/cig09.pdf
* Overview of artificial intelligence strategies for texas hold'em poker: http://poker-ai.org/archive/pokerai.org/public/aith.pdf
* New York Times article on computer poker: http://www.nytimes.com/2013/09/08/magazine/poker-computer.html?_r=0
