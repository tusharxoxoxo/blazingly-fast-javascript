In the JS game, what all things that would be slow

I have written alot, in this repo, how it make sense if i try to visit this branch in future and learn from this

1. creating js objects
2. network -> but here we are trying to improve javascript, we gonna pretend network is unimprovable, we not gonna worry 
about the latency of the users (ignore -> edge deploying, servers closest to groups of people, organizing web sockets with relation
from where are they coming  
3. promises 

all this is correct, but the best way to find out something is slow is by measuring it 

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
-g 100000 -> We are going to play a whole bunch of games
-t 2 -> time between connections, every connection to the server will be created every 2 seconds, so we have a better 
distribution, as supposed to make 500 instantly, those 500 will be created in a spread out manner 

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

we will want to refer to these later 

tsc && node dist/src/server.js --logPath /tmp/m-500
cargo run --release -- -q 500 -g 100000 -t 2 
and replace /tmp/testing  -> /tmp/m-500

![Screenshot 2024-01-31 at 11 28 43 AM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/7cfc94b7-c33e-4ec2-8477-41ad15cca568)
![Screenshot 2024-01-31 at 11 29 13 AM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/9d32fd49-b6d9-4968-989c-f5bfe9a1b670)
![Screenshot 2024-01-31 at 11 29 31 AM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/2267469a-2ca5-4763-a74c-4a9ec385d35a)


we gonna let this get 1-2 million ticks, then repeat it with 1000 and then 2000 
be careful, if u run the same log file twice u gonna get mixed results 

![Screenshot 2024-01-31 at 11 31 29 AM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/e7f5c17f-ef4f-4a0c-a2de-97d557cba342)
![Screenshot 2024-01-31 at 11 30 56 AM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/bb0ad64b-24e8-4252-aaae-8d55af7fec5e)

it is looking bad, performance is decreasing as we increase the number of servers 

It is a very crude step ladder test, u create a series of tests to see how this thing performs as we increase inputs
we give us a good idea of how our server is gonna do as it gets more complex 

very good to do multiple measurements

1st optimisation
hotspot optimization 
u simply look for the worst offender and try to fix that 
this isn't always the best way to go about things, but it's pretty simple  

launching the program, but with --inspect 
tsc && node --inspect dist/src/server.js --logPath /tmp/testing 
creating a new path, we don't wanna dirty our previous paths 

go to Chrome's performance tab 

the difference between deepmerge and assignment 
assign does key level assignment, so if u have an object that has A, B, and C and no matter whatever on the right-hand side
of those objects, it just simply assigns A, B, C
but deepmerge will go hey what are u, u are a string, we going to assign, u are an object, what are your keys, u are an array
Let go through every single value of your array, deeply lock the object, and try to merge it over one by one, take two 
different shapes of objects and make them into one, giving the right-most object the highest priority, that's a lot more work
then simply assigning, obviously much more expensive, deepmerge is done in javascript and object assign is a C++ method
if u can avoid it, don't do it 


go to chrome://inspect
go to Performace, do the profiling, and see the flame graph, u don't need a lot of data in the Performace tab, 
because it doesn't create a flame graph, it creates a timeline of what's happening 

so this is everything that is happening, every single call that is going up and down, there is some fudge factor they do for 
speed purposes, but this is what u get, 
tick the memory, 
find something that is eating a lot, idle time
click on the dark blue kind of area, go to bottom-up, zoom out, and will get the complete picture 
this is going to organize the function by name whose had the most self-time, 
there are two times with something, total time and self-time 

![Screenshot 2024-01-31 at 3 45 27 PM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/b28e1716-2c46-4efe-a44a-472cd02fab64)

the flat top is the time where it is spent, 
here I have added one image, in this image u can see there are a lot of things, but most of them get executed in no time
then the thing that is at the top that is taking most of the time, whatever is at the top, is the thing that is running 

![Screenshot 2024-01-31 at 3 52 13 PM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/4ca169b2-5e04-4d09-9a99-fb8d585c32d6)
![Screenshot 2024-01-31 at 3 49 38 PM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/fb56c02f-3e3f-4e55-8281-023dd20fa97b)


now let's look at the chart 
total time -> is the amount of time that appears in this chart 
self-time -> is the amount of time at the top of the chart, actually executing 

![Screenshot 2024-01-31 at 3 41 44 PM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/34a0db72-55bb-41dc-b3b3-830862fc45f1)
![Screenshot 2024-01-31 at 3 40 44 PM](https://github.com/tusharxoxoxo/blazingly-fast-javascript/assets/79051850/bc6d2de2-cbde-4766-bae6-de1896aa26ed)


the first thing I see is we are largely idle, that's bad, that's not what we want to see 
second, we spend a lot of time in the update method, which sucks 
In Prime's one, there is garbage collection, which I can't see in my one, that's strange, I think I made some mistakes 
while setting up my system 

if we click on the function links given, we can go to that function and see, where it is spending its time, but prime doubts, whether 
it's correct or not 


let's look at the code and try to fix it 

src/game/game.ts 
update function 

it's a little strange 
maps and sets are not always the right answer 

quote 
"And yes, my recommendation is to use std::vector by default. More generally, use a continuous representation unless 
there is a good reason not to." -Bjarne Stroustrup, creator of C++

here we are using sets because it is easy to add and remove 
but the problem is O(1) is a lie sort of, that constant has a big c in front of it, with an array, we have our memory nice
and packed in a single location but in the case of the set who knows where it all spread out 

let's try to prove our intuition, sets, and maps are slower than arrays
there is a breakpoint where we should use a set after which sets start to perform better than arrays, always check 

