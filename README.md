# Why 
- wanted to leverage lambda in a self hosted manner

# Why webasmm
- well after i discovered docker i LOVED the tool but i couldnt bring myself to ignore the level of bloat
in docker, like why do you need to run a whole os to run a binary (to be honest we have come quite a long way
to make the docker imgs as optimized as possible but at the end of the day you are still more or less running
a whole os) but i didnt really bothered with finding a solution until i started working jvm langs and saw a
pattern there and from there i had one in mind of how it a more efficient docker would look like. Then I
stumbled upon webasm and i wasa fascinated by the oportunites -> you can run any piece of code on a vm matching \
native performance

# Advantages
- the biggest advantage compared to lambda is peformance, although firecracker is a fantastic peice of tech
it is still dependant on containers so we run in the same problem
- ease of development you can ask anyone that has worked with lambdas that they are a bit unintuitive -> you have
  zip your code in a certain way that changes from lang to lang, testing them is hard, you cant run them locally
in a way that mmimicks the real thing enough (or atleast i couldnt find one and i searched quite a bit)
- the core of the service is webasm which has huge community backups so it will continue to be improved by
the community, which unlike firecracker although open source is really worked by aws employees 

# Addressing potential concerns 
## Lack of Layers
- well it is true that we lack layers and there is not really a good, performant and easy way to implement them
but layers are not a perfect strategy either. If we have soend a bit of time with lambda we eill see just how
error prone they are (mismatching versions of a dependency causing problems which leads to just packaging the
dependecies with nultiple versions and having multiple versions of the same layer for the sake of "cleaness")
it really starts to look like we are just uploading a standalone code (as from the start) but with extra steps.
In fact its so error prone that the standard has become to just provide images which handles all the dependcies
so that we dont have to posion ourselfs with layers which just demonstrates my point that they are evil.
So to answer with one sentence - yes we doint support layers, but the community doesnt seem to like them either





# How It Works
## Architecture
We took a lot from the aws lambda architecture but it basically functions in the foillowing way 
![image](https://github.com/user-attachments/assets/8817cf86-bc8e-4bdd-a579-5ed84072369e)


So when we send a req to invokde a certain lambda we first hit a server which decides whether to spawn a new instance to handle the req or forward it to an existing one (there are three options for configuring this, see routing options for details) from there it sends the data that needs to be present for the handler (idk if we need to wrap each lambda handler in an api so that we can reuse instances) and the handler returns result which you recieve


## Scheduler 
The role of this piece of code is to decide what to do with a lmabda when it  has finished (when a lambda finfishes execution or is forcefully stopped more details or lambda handlers internals below) it has two three options -> keep alive, kill or invoke
- keep alive: it tells the handler to be idle for `x` amount of ms and when it finishes it again asks the scheduler for what to do (design decision - you should decide whether lambdas will supervise themselfs e,g, you run the code as a subprocess of a process which takes care of all the other things and just operates based on messages from the scheduler or the scheduler kills them )
-  kill: it kills the instance
-  invoke: it invokes the lambda again

## Lambda Handler internals 
Basically every node has uploaded code which starts automatically which spawns a grpc server which has an endpoint called `do` which can accept one of the three scheduler operations and decided what to do, also before being able to accept requests the server fetches the code for its handler (this shouldnt be a concern since no one should send requests to the handler). 

Code fetching logic: to avoid the need of making dynamic code genration to put in the lambda e,g, to avoid the need of a preprocessor that we replace or put a string as code (ask me to explain it again) we have the following startagy a lambda will reach to a remote server from which it will grab the code as a file and run it

example code
```rs 
def main():
  file = getFile()
  runFileAsWebAsmHAndler(file)
  loop:
    acceptCommand();
  


```



## Routing options
There are three options:
- cuaotm: Upload your code which decides whether to send it to an existing instance or spawn a new one (you get access to how much time  has passed since each lambda handler has started executing in ms in the function arg)
- default: use our algo for deciding 
- ai: we have an ai which has access to the data you send to a lambda handler and the time a lambda needed to complete and from this data it learns based on the data supplied and all the current executions times of lambdas whether it is better to wait for a lambda to finish ir spawn a new instance (e.g. it checks if the cold start of a lambda is worth it over just waiting for an already started lambda to finsih to deliever the best outcome). Another metrics to which the ai has access to is your traffic petterns e.g. by timestamp how much much requests you have recieved at this time (this too helps for deciding if it is better to start a lambda since it ousl be bad for the ai to wait a few times introducing a hrottle annd from there picking up)
