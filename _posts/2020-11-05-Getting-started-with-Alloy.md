# What is Alloy
Alloy is a language that can be used to describe models and is also a tool that is used to explore these models. 
### Where to download
The latest build of Alloy can be found [here](https://alloytools.org/download.html)


## Flow control app
What we are going to model is a smart sink. This smart sink will have the ability to be filled up via a mobile application.

The sink has a water valve, a drain valve, and a water level sensor which are all connected to a wifi network.

The mobile app is connected to the internet and has a fill and drain button.

#### Here is the defined behavior of the mobile app

Fill button is clicked ->  
* Close drain valve  
* Open water valve

Drain button is clicked ->  
* Open drain valve  
* Close water valve

Water sensor is triggered ->  
* Close water valve

#### Here is the behavior of the sink

App says to close water valve ->  
* Close Water valve

App says to close drain valve ->  
* Close drain valve

App says to open water valve ->  
* Open water valve

App says to open drain valve ->  
* Open drain valve 


#### Why are we trying to model this?
What we want to avoid is the sink overflowing. So what we are going to try and see is that if at any point during operation if there is an internet issue, would the sink overflow?

#### How do we model this?
Let's start with the describing the sink and the app.

Both will have internal properties:
* WaterValve - {Open or Closed}
* DrainValve - {Open or Closed}
* WaterSensorTriggered - {True or False}
* InternetWorking - {True or False}

And the sink with have another value of
* WaterLevel - {Empty, Filling, Full, OF}


Let's create some [signatures](http://alloytools.org/tutorials/online/sidenote-format-sig.html) for the them!

Starting with an empty signatures
```
one sig App {
}

one sig Sink {
}
```

Now we need to add some fields to the signature

Remeber we have three properties that we want to represent:  
* WaterValve
* DrainValve
* WaterSensorTriggered
* InternetWorking

and an additional one for sing
* WaterLevel

Both Valves can have a value of open or closed and the Sensor can have a value of true or false.
And let Water Level have a value of {Empty, Filling, Full, OF}

We can define the types by creating some enums
```
enum Valve {Open, Closed}
enum Boolean {True, False}

enum WaterLevel {Empty, Filling, Full, OF}
```

Then adding these fields to our signature, we get
```
one sig App {
waterValve: Valve one,
drainValve: Valve one,
waterSensorTriggered: Boolean one,
internetWorking: Boolean one
}

one sig App {
waterValve: Valve one,
drainValve: Valve one,
waterSensorTriggered: Boolean one,
internetWorking: Boolean,
waterLevel: WaterLevel
}

```

The `one` following each field means that the signature only has one of these fields.

A thing to remeber here, these fields we defined are a state of our system. These states have a relationship with time and we need to represent this.

###### This might me hard to understand, you need to remeber that we are not writing a program, we are creating a model of and need to model the states and the transitions of our system.

To do this, we add the relationship symbol `->` and the thing we have a relationship with `Time`.

```
one sig  App {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time
}

one sig  Sink {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time,
waterLevel: WaterLevel one -> Time
}
``` 
Now we need to define the state transitions

#### Sink
Remeber, all of our fields are a relationship on time, so each unit of time has the above properties


Now our sink water and drain valves should reflect the state given by the app, but only if the internet is working for both. To reflect that, we can write 

```
one sig  Sink {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time,
waterLevel: WaterLevel one -> Time
}{
	all t: Time {
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (waterValve.t = App.waterValve.t)
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (drainValve.t = App.drainValve.t)
	}
}
```

Now the water level needs to be able to change state, we can do that by
```
one sig  Sink {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time,
waterLevel: WaterLevel one -> Time
}{
	all t: Time {
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (waterValve.t = App.waterValve.t)
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (drainValve.t = App.drainValve.t)
	}
	all t: Time - last |
		let t' = t.next {
			(waterValve.t = Open and drainValve.t = Open) 		implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Closed and drainValve.t = Closed) 	implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Open and drainValve.t = Closed) 	implies (waterLevel.t' = increaseLevel[waterLevel.t])
			(waterValve.t = Closed and drainValve.t = Open) 	implies (waterLevel.t' = decreaseLevel[waterLevel.t])
		}
}
```

and of course, we need to add the state transition for the sensor

```
one sig  Sink {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time,
waterLevel: WaterLevel one -> Time
}{
	all t: Time {
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (waterValve.t = App.waterValve.t)
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (drainValve.t = App.drainValve.t)
	}
	all t: Time - last |
		let t' = t.next {
			(waterValve.t = Open and drainValve.t = Open) 		implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Closed and drainValve.t = Closed) 	implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Open and drainValve.t = Closed) 	implies (waterLevel.t' = increaseLevel[waterLevel.t])
			(waterValve.t = Closed and drainValve.t = Open) 	implies (waterLevel.t' = decreaseLevel[waterLevel.t])
			waterSensorTriggered.t' = True iff waterLevel.t = Full
			waterSensorTriggered.t' = False iff waterLevel.t != Full
		}
}
```

You might of notices we added the following functions `increaseLevel` and `decreaseLevel`. All these do is increment or decrement the water level. We are going to create those

```
fun increaseLevel: WaterLevel -> WaterLevel {
	waterLevelOrder + OF -> OF
}

fun decreaseLevel: WaterLevel -> WaterLevel {
	~waterLevelOrder + Empty -> Empty
}

fun waterLevelOrder: WaterLevel -> WaterLevel {
	Empty -> Low +
	Low -> Half +
	Half -> High +
	High -> Full +
	Full -> OF
}
```

Now we are going to add the state transitions for the App. For sake of ease, we are going to say make the app prtty simple.

The water valve is open iff the water sensor is not triggered  
The drain valve is open iff the water valve is closed  
and vice-versa

We also need to add a transition for the water sensor, remeber, the app is getting this from the sing. Which will only work if both have internet.

```
one sig  App {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time
}{
	all t: Time {
		(internetWorking.t = True and Sink.internetWorking.t = True) implies (waterSensorTriggered.t = Sink.waterSensorTriggered.t)
	}
	all t: Time - last |
		let t' = t.next {
			waterValve.t' = Open iff 
			(
				waterSensorTriggered.t = False 
			)
			waterValve.t' = Closed iff 
			(
				waterSensorTriggered.t = True
			)
			drainValve.t' = Open iff 
			(
				waterValve.t = Closed
			)
			drainValve.t' = Closed iff 
			(
				waterValve.t = Open
			)
		}
}
```

Now we have to set the initial states

```
fact init {
	App.waterValve.first = Closed
	App.drainValve.first = Open
	Sink.waterLevel.first = Empty
}
```

and add the predicate to determine if the water overflows

```
pred overflow[t: Time] {
	let t0 = t {
		some t0
		Sink.waterLevel.(t0) = OF
	}	
}
```
This is saying that there exists some uint of time where the water level is OF

We have been using `Time` through out the specification, but what is Time? It is nothing more than a signature

```
sig Time {}
``` 

Now we can add the command to run this
```
run overflowTest {
    some t : Time | overflow[t]
}for 1 but 20 Time
```


All in all, we have

```
open util/ordering[Time]
enum Valve {Open, Closed}
enum Boolean {True, False}
enum WaterLevel {Empty, Low, Half, High, Full, OF}

sig Time {}

fact init {
	App.waterValve.first = Closed
	App.drainValve.first = Open
	Sink.waterLevel.first = Empty
}

pred overflow[t: Time] {
	let t0 = t {
		some t0
		Sink.waterLevel.(t0) = OF
	}	
}

run overflowTest {
    some t : Time | overflow[t]
}for 1 but 200 Time


one sig  App  {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time
}{
	all t: Time {
		(internetWorking.t = True and Sink.internetWorking.t = True) implies (waterSensorTriggered.t = Sink.waterSensorTriggered.t)
	}
	all t: Time - last |
		let t' = t.next {
			waterValve.t' = Open iff 
			(
				waterSensorTriggered.t = False 
			)
			waterValve.t' = Closed iff 
			(
				waterSensorTriggered.t = True
			)
			drainValve.t' = Open iff 
			(
				waterValve.t = Closed
			)
			drainValve.t' = Closed iff 
			(
				waterValve.t = Open
			)
		}
}

one sig  Sink {
waterValve: Valve one -> Time,
drainValve: Valve one -> Time,
waterSensorTriggered: Boolean one -> Time,
internetWorking: Boolean one -> Time,
waterLevel: WaterLevel one -> Time
}{
	all t: Time {
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (waterValve.t = App.waterValve.t)
		(internetWorking.t = True and App.internetWorking.t = True) 	implies (drainValve.t = App.drainValve.t)
	}
	all t: Time - last |
		let t' = t.next {
			(waterValve.t = Open and drainValve.t = Open) 		implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Closed and drainValve.t = Closed) 	implies (waterLevel.t' = waterLevel.t)
			(waterValve.t = Open and drainValve.t = Closed) 	implies (waterLevel.t' = increaseLevel[waterLevel.t])
			(waterValve.t = Closed and drainValve.t = Open) 	implies (waterLevel.t' = decreaseLevel[waterLevel.t])
			waterSensorTriggered.t' = True iff waterLevel.t = Full
			waterSensorTriggered.t' = False iff waterLevel.t != Full
		}
}


fun increaseLevel: WaterLevel -> WaterLevel {
	waterLevelOrder + OF -> OF
}

fun decreaseLevel: WaterLevel -> WaterLevel {
	~waterLevelOrder + Empty -> Empty
}

fun waterLevelOrder: WaterLevel -> WaterLevel {
	Empty -> Low +
	Low -> Half +
	Half -> High +
	High -> Full +
	Full -> OF
}


```

If you execute this in Alloy, you will see that an *Instance* is found, or something that causes our predicate to be true.