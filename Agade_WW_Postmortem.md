# Agade Wondev Woman Postmortem

I will describe my AI which finished second in the Wondev Woman contest.

## Introduction

Like many others my first thought when reading the rules of the game was the classic algorithm of minimax. I was so unable to think of anything else that I got out of Wood 1 with a minimax. As usual I wrote a [referee](https://github.com/Agade09/CG-WW-Arena) program to be able to play games between versions of my code and measure improvements.

## Locating the enemy in the fog

To be able to minimax you need to place the enemy somewhere on the map, and the enemy's position is not always known because the game had a fog of war according to which you could only see a distance of 1 away from your pawns. Nevertheless the terrain is fully known to you. The goal is then to extract maximal information from what is passed to you, reducing his possible locations as much as possible and then picking one of the possibilities.

In my experience having a good system to locate the enemy pawns in the fog is worth massive gains in win rate. I had 85% win rate against my previous version when I went from removing hidden enemy pawns from the board to placing them at a random possible location.

### Possible location deduction

If you see square (x,y) get built up you know that the enemy has a pawn a distance of 1 away from it, in 8 possible locations. But if you know that the enemy previously was to the right of that square in (x+2,y) then you can narrow down to 3 possible locations: (x+1,y) (x+1,y+1) and (x+1,y-1). So taking into account the past as well as the present is important in order to extract maximal information. In the following paragraphs I will describe what I believe to be the optimal way of using all available information.

Consider your enemy's possible locations as a set<array<vec,2>> where set is the data structure that doesn't allow duplicate elements and vec is a position vector such as (1,2). On your first turn initialise the set with all possible pairs of locations that are coherent with the current state: visible pawns are where they should be, hidden pawns are more than 1 distance away from your pawns. On each following turn, for every possible location pair of the enemy pawns, take the state of the game on your previous turn, place the enemy pawns in that location pair, simulate the move that you made and then simulate every possible move the enemy could have made from there, if any state is coherent then the enemy's pawns could now be there. In pseudocode:

```
set<array<vec,2>> Possible_Locations(state Previous_state,state S_observed,set<array<vec,2>> Previous_Possible_Locations,action my_last_move){
    set<array<vec,2>> Possible_Locations;
    for(array<vec,2> Loc:Previous_Possible_Locations){
        state S=Previous_state;
        S.PlaceEnemies(Loc);
        S.Simulate(my_last_move);//The enemy pawns need to be placed before this because they might block your build/fail your push
        vector<action> Moves=S.LegalMoves();
        if(Moves.size()>0){
            for(action mv:Moves){
                S.Simulate(mv);
                if(Coherent(S,S_observed)){
                    Possible_Locations.insert(array<vec,2>{S.EnemyPawns[0],S.EnemyPawns[1]});
                }
            }
        }
        else{
            if(Coherent(S,S_observed)){//He could have been in this position and died because he had no moves
                Possible_Locations.insert(array<vec,2>{S.EnemyPawns[0],S.EnemyPawns[1]});
            }
        }
        
    }
    return Possible_Locations;
}
```

I believe you cannot narrow down the enemy's possible locations more than this. This bruteforce strategy is robust and is only time consuming on the very first turns when the enemy has ~400 possible location pairs in the fog. The worst case was about 2.5ms on turn 2 for my code.

#### Tracking the score

The current score is also not passed as input to your AI. It would be useful to have it in order to determine guaranteed wins in the minimax. I think it would be possible to have a good idea of the enemy's score using this approach but I suspect that there would be many situations where you just wouldn't be able to be sure what the score is because you wouldn't always be able to know exactly what path the enemy pawn took in the fog. I chose not to bother with tracking the score. In my minimax the score always starts at 0 and I count scored points, not total points.

### Placing the enemy

After you have narrowed down the enemy's possible locations as much as possible you now have to pick from the available possibilities and place the enemy there. Many times there will only be one possibility, but in the general case you have to make a decision between several. My choice was in order of preference:

* Place the enemy at the location where he would be if he had played the previous turn's principal variation, that is to say, the best move I found for my opponent in my minimax search. 
* If the principal variation is not possible according to the observed state then place the enemy in the worst location for me according to my evaluation function.

### Player 1

If you are player 1(red) you can exploit the fact that your opponent moved first to narrow down his possible location pairs. You cannot know that you are player 1 for sure though as your opponent could have made a move which did not successfully build on a cell like a failed push or a move without a build. If at turn 1 you see a cell with a height of 1 you can be sure that you are player 1 though. If you are sure to be player 1 you can then, instead of just assuming that your enemy could be anywhere that's coherent with the observed state, run a bruteforce search where for everywhere you could have spawned (you don't know where you spawned the enemy could have pushed you on his first turn) and for every position the enemy could have spawned in, you test all his moves and if a state is coherent with the observed state you add the enemy position pair as a possibility. It's turn 1 you have 1s anyway, and knowing where your enemy is is worth alot of elo in this game.

### Details

#### Considering possible enemy locations as pairs

It is important to consider the enemy's possible locations as pairs and not possible locations for each pawn. If you see that the enemy has built up a cell and both his pawns could have done it, a pawn-wise approach would add possible locations for both pawns in effect consider 4 possibilities: "pawn 0 moved", "pawn 1 moved", "pawn 0 and pawn 1 moved" and "no pawns moved". The pair-wise approach only considers:  "pawn 0 moved" and "pawn 1 moved".

#### No possible locations

If you have no bugs the algorithm I describe will successfully identify all possible locations of the enemy and you should never encounter a situation where the enemy has no possible location pairs. Except! if he crashes/surrenders with available moves. In my code if there are no available moves I mark the enemy as dead (skips his turn in the minimax search) and look for his possible locations again with the new information of his death. Sometimes this second search fails as well because the enemy actually might have died several turns ago and my location pairs are therefore all false because I was assuming that he was moving every turn. If this happens I wipe the possible location pairs and refill it from scratch according to the turn 1 method of placing him everywhere he could be, visible pawns being where they should be and hidden pawns being in the fog. I think in principle you could save every turn's possible location pairs and go back in time one turn at a time assuming he died earlier. But I didn't think it was really important to be so precise against crashing opponents.

## Evaluation

### Voronoi

When I look at replays it seems to me like 90% of games are won by pushing the enemy into a much smaller area and crushing him in terms of score. Pushing the enemy into a smaller area than yours is achieved by the Voronoi heuristic which involves counting the cells you can reach before your opponent (correctly assuming whoever's turn it is moves first).

It seems very rare to me that the game finishes with a tight score line where a precise evaluation of zone heights and topology would be useful. On top of that I soon found out that performance was very important in this game. I found a rule of thumb of 10% performance <=> 4% win rate. I have never noticed performance having such a dramatic effect on Codingame before. So I was really doubtful that it would be wise to waste precious clock cycles with a precise evaluation when 90% of games can be won by out-voronoying your opponent. Hence, for 9 days my evaluation function was the score difference plus twice the Voronoi difference (you can maybe get about 2 points per square in the endgame).

```
voronoi_difference=voronoi[0]-voronoi[1];
score_difference=score[0]-score[1];
```

### Accessible neighbours

On the last Sunday I gave evaluation some more thought. To me there was only one problem with the Voronoi heuristic which is that it is collective between my pawns. I felt that my evaluation didn't really give any incentive to keeping pawns alive for equal voronoi. So I gave +1 point per neighboring cell each pawn could move to, assuming the cell is unoccupied. This helped win rate alot and made me competitive against T1024 for the first time in days. There was a performance cost but it was worth it.

### Final evaluation

```
2*voronoi_difference+score_difference+accessible_neighbours_difference
```

I feel like I probably missed alot of elo on the evaluation side of things, I hardly tried anything and I barely tried tweaking coefficients. But I don't really doubt my judgment of spending so much time improving search depth, I wasn't wrong about it, a deep minimax with a Voronoi evaluation makes you very strong.

## Improving search depth

As I mentioned I believe search depth is key in this game. I don't think one can win the pushing game by evaluating the board in some genius way, brute force is required. I achieved a depth of 5 at the beginning of the game, increasing to 6 and 7 as the map size decreased and going way past 10 in many end game situations (where maybe it didn't matter anymore). In this section I will mention different techniques to improve your search depth.

### Caching the evaluation function

The evaluation function, especially the Voronoi was the most computationally expensive part of the code. So caching it was helpful. To do this you have to hash the state to a number so you can use that hash as a key in a map. I used the [Zobrist hash](https://chessprogramming.wikispaces.com/Zobrist+Hashing). This hash does the job and can be updated in a few operations in your Do/Undo functions.

### Pawn permutation symmetry

In the game your pawns have ids and because the moves are given by pawn id, for example "MOVE&BUILD 0 N N", it is natural to write your action structure as something like this:

```
struct action{
    int type;//0: MOVE 1:PUSH
    int id;
    vec to,build;//build is the location of the build/push target
}
```

But in the game it really does not matter if you swap your pawns 0 and 1. It literally doesn't change a thing except for outputting to the referee program. For this reason I changed my action structure to something like

```
struct action{
    int type;//0: MOVE 1:PUSH
    vec from,to,build;//build is the location of the build/push target
}
```

along with a few other things in the code in order to reflect this symmetry. This allows for more efficient caching as states identical with respect to this pawn permutation symmetry will properly hash to the same thing.

### Iterative deepening

In order to use your allocated time you either search to a fixed depth low enough that you never time out, or use [iterative deepening](https://chessprogramming.wikispaces.com/Iterative+Deepening) whereby you minimax from depth 1 upwards until you run out of time, raise an exception and keep the result of the last completed search. It may seem that you are wasting computation by calculating lower depths but this is minimal as the cost of the previous depths is much smaller. On top of that, as we will see, we can use the results of lower depth searches to order moves better in the current search. The uncompleted search will also start filling up the Evaluation function hash table which might speed up the next turn's Minimax if the principal variation is followed by the enemy.

### Performance

The faster your code the deeper you can search, which means you have to code in C/C++ and in the case of C++ add some pragmas at the top of your code such as

```
#pragma GCC optimize("-O3,inline,omit-frame-pointer,unroll-loops")
```

in order to get closer to a proper -O3 compilation of your code.

I wrote my first versions in "clean" C++ standard library style but I then had to add ugly performance hacks here and there. For example in my Voronoi function I was allocating a boolean table to mark cells as visited. Memory allocation takes time so I used a silly global variable instead. And you can avoid memsetting the visited table to false by making it an integer table filled with 0s on the first turn and using a global variable vor_counter which starts at 1 and goes up at every Voronoi(), a cell is visited if it is set to vor_counter and not visited otherwise. Instead of a memory allocating std::queue I had another global variable array with pointers to the beginning and end of that "queue". I also gained performance by caching things like a cell's accessible neighboring cells and which pawn is occupying a cell (-1 if no pawn). I had to represent moves as integer indexes of a global array, etc... Performance optimisation was kind of boring but worth alot of elo.

I don't think I had the best performance, there's probably some margin there. On the first turn, in one second I would explore ~1 million minimax nodes (all nodes not juste leaf nodes). Of course this is not comparable to your performance if your evaluation function is more complicated than mine.

### Eliminate moves

T1024 and reCurse did this, but I didn't. The game felt so bruteforcy to me that I really couldn't imagine how to heuristically know that a move wouldn't be good. Maybe I missed out on alot of elo here.

### Alpha beta

Alot of you know about [alpha beta pruning](https://en.wikipedia.org/wiki/Alpha%E2%80%93beta_pruning), it's very basic and easy to add to your minimax. It reduces tree size considerably and gives you 50% more depth in the average case and doubles your search depth in the best case.

### Move ordering

In order to reach the depth doubling best case of minimax whenever a node is explored the best move needs to be explored first. Of course this will never be reached in practice because if we already knew the best move we would not be minimaxing to find said best move. But we can get closer to that situation and reach deeper search depths by ordering more promising moves first and therefore causing more alpha beta cutoffs to prune down the tree. [Move ordering](https://chessprogramming.wikispaces.com/Move+Ordering) is a serious topic in chess programming because it can greatly improve depth. 

I wrote code specifically to measure the effectiveness of my move ordering. I took the part of my referee program that generates maps and wrote a function which generates maps and runs a Minimax search to a fixed depth over and over, computing the average size of the tree along the way. This allowed me to measure ordering improvements and every time I improved this metric my AI indeed seemed to improve. The only problem I see is that I am only testing my move ordering in a fresh spawn situation where ordering might be very different than, say, at the end game.

To order moves I used in order of priority the [hash move](https://chessprogramming.wikispaces.com/Hash+Move), the [killer heuristic](https://chessprogramming.wikispaces.com/Killer+Move), the [history heuristic](https://chessprogramming.wikispaces.com/History+Heuristic) and a game specific height heuristic. This improved the win rate considerably.

#### Hash move

At every minimax node I store in a hash table the best move found, so either the best move or the move that caused an alpha beta cutoff. If I ever visit that node again I try that move first.

#### Killer heuristic

The minimax function passes along a table keeping track of the last two different moves that caused an alpha beta cutoff at each depth. If the first/second slot of the killer heuristic is a legal move try that move first.

#### History heuristic

The minimax function passes around a table of size 2*W^6 where W is the width of the square grid. Every possible move is indexed as

```
action_idx=type*W^6+from.idx()*W^4+to.idx()*W^2+build.idx();
vec::idx(){//Gives a unique integer id to a position vector (x,y)
    return y*W+x;
}
```

The table starts of with zeros at the beginning of the search and at every minimax node the best move/the move that caused a cutoff gets it's history value incremented by `2^depth`(`1<<depth`) where depth is for example 3 at the root of the minimax and 0 at the leaves where you evaluate. When sorting the moves, a move with a higher history value is played first.

#### Game specific heuristic

In case of a tie in history values I sort by the height of the action.to cell, favoring moves that go higher/push the enemy from higher.

#### Performance

I was actually spending quite alot of time sorting all these move vectors but I found I was able to get rid of alot of that overhead by only sorting moves if the hash move and the killer moves hadn't caused a cutoff.

### Negascout

Once you have a pretty good move ordering, to further prune the search tree you can implement a variations of the classical minimax/negamax algorithm such as [Negascout](https://chessprogramming.wikispaces.com/NegaScout). In the very rough way that I understand it, the idea of those algorithms is that your move ordering is so good that you're most of the time going to be trying the best move first, so you just assume you're correct and from time to time you have to re-search a part of the tree because you were wrong. These algorithms are correct, they will always return the same result as a minimax, and they will always search less unique nodes than a standard minimax as long as that assumption of testing the best move first is true from time to time. However if your move ordering isn't good enough and/or you're not caching enough things the performance overhead of the re-searches can outweigh the pruning gains. If you try Negascout, make sure to test that your AI actually got stronger.

In pseudocode instead of exploring your nodes as

```
score=-negamax(state,depth-1,-beta,-alpha,-color);
```

you explore your nodes as:

```
if(first_move){
    score=-Negascout(state,depth-1,-beta,-alpha,-color);
}
else{
    score=-Negascout(state,depth-1,-alpha-NullWindowSize,-alpha,-color);
    if(score>alpha && score<beta && depth>1){
        double score2=-Negascout(state,depth-1,-beta,-score,-color);
        score=max(score,score2);
    }
}
```

where NullWindowSize is a constant such that your evaluation function can't return anything between `-alpha` and `-alpha-NullWindowSize`. Negascout assumes that the first move is the best one and tries to prove the other moves aren't as good with [null window searches](https://chessprogramming.wikispaces.com/Null+Window).

## Details

#### Don't remove permutations in enemy location pairs

If, like me, you were tempted to consider the enemy location pairs {(0,1),(2,3)} and {(2,3),(0,1)} as identical, don't. There are slightly rare cases where you can eliminate possibilities by knowing which pawn id is where. The referee gives you as inputs not only visibility information but information on which id is visible. So alot of my code is symmetric to id swapping but this isn't.

## Acknowledgments 

Magus and Neumann for performance tips. Usually performance doesn't matter so I was really rusty.

If I remember correctly on Monday morning I woke up to find reCurse farming me. Played two games in IDE, saw him shoving me in corners => Todo: Delete BFS space eval, write Voronoi. I was working on my fog detection and hadn't realised that would be the meta yet.

## Conclusion

I really liked the simplicity of the game. It wasn't particularly exciting to optimise performance. And I guess the game wasn't heuristic-friendly at all. But it was nice to visit this classic algorithm of minimax and try the different techniques to reduce the size of the search tree. Trying to place your opponent despite the fog was also an interesting problem.

Feels like I got beaten by T1024 on a coin flip (0.11 elo...). But I'm very happy to have made a fourth podium with a strong AI.