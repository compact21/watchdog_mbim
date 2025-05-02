# difference between "exit", "break", "continue", "return" into loop

just for my future reference (as I said I'm not a good programmer)

run test_simple (without interruptions only for testing)
<code>
0
start while
0
1
2
3
4
5
6
7
8
9
10
stop while
11
</code>

run test_break
<code>
0
start while
0
1
2
3
4
5
6
7
stop while
7
</code>

run test_exit
<code>
0
start while
0
1
2
3
4
5
6
7
</code>

run test_return
<code>
0
start while
0
1
2
3
4
5
6
7
</code>

run test_continue (<b>warning attention it will go on forever until it is killed</b>)
<code>
start while
0
1
2
3
4
5
6
7
7
7
7
7
7
7
7
7
CTRL+C
</code>
