In the js game what all things would be slow

1. creating js objects
2. network -> but here we are trying to improve javascript, we gonna pretend network is unimprovable, we not gonna worry 
about the latency of the users (ignore -> edge deploying, servers closest to groups of people, organising websockets with relation
from where are they coming  
3. promises 

all this is correct, but the best way to find out something is slow, is by measuring it 

commands
inside shooter

server 
tsc && node dist/src/server.js --logPath /tmp/testing 
build node with tsc
watch command is missing, we want it to be running some of the time, not all the time 
have also set a log path 

client
cargo run --release -- -q 500 -g 100000 -t 2
run in the release mode 
-q 500 -> 500 connections created to the server 
-g 100000 -> we are going to play a whole bunch of games
-t 2 -> time between connections, every connections to the server will be created every 2 seconds, so we have a better 
distribution, as suppose to make 500 instantly, those 500 will created in a spread out manner 

viz server 
go run cmd/main.go
echo server in go and htmx 
go to the localhost
/tmp/testing


we have something
now we are going to make some changes 

small step ladder approach
run at 500 concurrent games 
run at 1000 concurrent games 
run at 2000 concurrent games 

to save separate log files of each of them
--logFile /tmp/master -500
--logFile /tmp/master -1000
--logFile /tmp/master -2000

we will want to refer these latter 

tsc && node dist/src/server.js --logPath /tmp/m-500
cargo run --release -- -q 500 -g 100000 -t 2 
and replace /tmp/testing  -> /tmp/m-500

we gonna let this get 1-2 million ticks, then repeat it with 1000 and then 2000 
be careful, if u run the same log file twice u gonna get mixed results 

it is looking bad, performace is decreasing as we increase the number of servers 

its a very crude step ladder test, u create a series of test to see how does this thing performe as we increase inputs
we give us good idea how our server is gonna do as it gets more complex 

very good to do multiple measurements

1st optimisation
hotspot optimisation 
u simply look for the worst offender and try to fix that 
this isn't always the best way for go about things, but it's pretty simple  

lauching the program, but with --inspect 
tsc && node --inspect dist/src/server.js --logPath /tmp/testing 
creating a new path, we don't wanna dirty our previous paths 

go to chrome's performace tab 

the difference between deepmerge and assignment 
assign does key level assignment, so if u have an object that has A, B and C and no matter whatever on the righthand side
of those objects, it just simply assigns A, B, C
but deepmerge will go hey what are u, u are a string, we gonna assin, oh u are an object, what are your keys, oh u are an array
lets go thorugh every single value of your array, deeply lock the object and try to merge it over one by one, take two 
different shape of object and make them into one, giving right most object with highest priority, that's alot more work
then simply assigning, obviosly much more expensive, deepmerge is done in javascript and object assign is an c++ method
if u can avoid, don't do it 


go to chrome://inspect
go to performace, do the profiling and see the flamegraph, u don't need alot of data in performace tab, 
because it doesn't create a flamegraph, it creates a timeline of what's happening 

so this is everything that is happen, every single call that is going up and down, there is some fudge factor they do for 
speed purposes, but this is what u get, 
tick the memory, 
find something that is eating alot, idle time
click on the dark blue kind of area, go to bottom-up, zoom out u will get the complete picture 
this is going to organise the function by name whose had the most self time, 
there are two times with something, totaltime and self-time 

the flat top is the time where it is spending, 
here i have added one image, in this image u can see there are alot of things, but most of them getting executed in no time
then the thing that is at the top that is taking most of time, whatever is at the top, is the thing that is running 

now let's look at the chart 
totaltime -> is the amout of time that appears in this chart 
self-time -> is the amount of time at the top of the chart, actually exceuting 

the first thing i see is we are largely idle, that's bad, that's not what we want to see 
second we spend alot of time in the update method, that sucks 
in prime's one, there is garbage collection, which i can't see in my one, that's strange, i think i made some mistakes 
while setting up my system 

if we click on the function links given, we can go that function and see, where it is spending it's time, but prime doubts, wheather 
it's correct or not 


lets look at the code and try to fix it 

src/game/game.ts 
update function 

it's a little strange 
maps and sets are not always the right answer 

quote 
"and yes, my recommendation is to use std::vector by default. More generally, use a contigenous representation unless 
there is a good reason not to." -Bjarne Stroustrup, creator of c++

here we are using sets because it is easy to add and remove 
but the problem is O(1) is a lie sort of, that constant have a big c in front of it, with array, we have our memory nice
and packed in a single location but incase of set who know where it all spread out 

lets try of prove our intiution, sets and maps are slower than arrays
there is a break point where we should use set after which sets starts to performe better than arrays, always check 

