# Let's Build A State Machine

---

Hello again, I'm back from the metaphorical dead to enthrall all of you with another exciting tutorial about the amazing topic of ✨**State Machines**✨!

I've just got done with the launch of my state machine framework [**Statement**](https://refreshertowel.github.io/docs/statement/), and so state machines have been on my mind a lot over the last few weeks.

---

## The State of States

So...What is a state machine? Well, consider a player character in a platformer. They might be able to run around, maybe climb ladders, possibly swim, perhaps even fly through the sky with a power up or something. Each of those “actions” could be considered a state, and a state machine is the thing that handles how you move in or out of those states, and deals with the actual working of the state as well.

Why would you want to build a whole state machine just to handle some actions like that? Let me explain, my dear reader, the pains of trying to implement stuff like that without a state machine. Firstly, you have your general movement. `x += hor_speed; y += ver_speed;`, etc. Simple and easy, no states required. Ah, but now you want to add climbing. You don't want your character to move horizontally on the ladder, so you decide “Ok, I'll just set an is_climbing flag to true when I start to climb, and then lock the horizontal movement behind a check to make sure we aren't climbing before it can run.

This seems reasonable, but it's the first dangerous step into heavy spaghetti-code territory. Next you try to add swimming, uh-oh, you'll need to add another flag to check for that, and then cancel out of the old movement completely in order to switch to the different movement for swimming (more floaty perhaps, with 360 degree motion). So you've now got three “sections” of code competing with each other, and being tied behind interlocking flags (what happens if you get on a ladder in the water?).

Now you add flying, and suddenly there's a whole new thing you need to check for and stuff is starting to get annoyingly complicated. Gameplay errors start creeping in (weird movement in certain situations, etc), and you're starting to bang your head on the desk wondering where it all went wrong.

This is the part of the story where the triumphant state machine walks into the room, head held high, to heroically save the day.

You see, the secret sauce behind state machines is they let you “carve out” little sections of code into “states” and then guarantee that only the code from one state is run at a time. It makes the states mutually exclusive, which then means, instead of having to try to figure out how your interlocking system of flags is leading to the player sprite incorrectly changing into swimming while you're on land, you can literally just reason about each small chunk of code at a time, safe in the knowledge that flying code will NEVER be run during swimming, no matter how hard your playtesters try to make that happen.

This might not actually sound all that amazing on the face of it, but once you start experiencing the ease of development this code partitioning allows for (and the many varied situations it can be helpful in), you'll wonder how you ever did anything any other way.

>This tutorial will build a very simple version of the same kind of system that Statement uses. If you don't have the time or expertise to follow this tutorial to get the simplified state machine running and then add all the necessary extensions and edge cases to it, then consider purchasing Statement. If you do, you'll get an easy to use state machine system with cool advanced features like:
>
>- Queued transitions and declarative transitions.
>- State stacks and history.
>- Payloads / non interruptible states.
>- Pause support, timers, and built in debug helpers.
>- And more. 
>If you want to check out the full feature list, check out the [Statement docs](https://refreshertowel.github.io/docs/statement/).
>For many people, the cost is more than worth the time it would take to develop, test, and maintain all this yourself.
{: .note}

---

## Starting States

So, the preamble is out of the way, and we can begin with an implementation.

First, let me say, there are many, many ways to implement a state machine, from if...else if...else chains (*ewwwww*) to switch statements (*slightly less ewwww but still...*) to setting a state variable to different functions and running that state variable (ok, but not very flexible and has some notable shortcomings) and more.

We'll be implementing my favoured method, which is to use a struct for the state machine and for each state, and having the whole thing revolve around a few simple methods called from either the state machine or the state.
So, since we'll be implementing a repeated pattern of struct creation for the state machine structs, we should -immediately- be thinking of a constructor. Let's build a very basic one now. Create a new script, call it `scr_state_machine` or something and put this code in it:

```js
function StateMachine() constructor {
	state = undefined;
}
```

Done and dusted. We call `sm = new StateMachine();` and we have a nice little struct stored in sm with a single member variable called `state`. Ok, I'll foreshadow that this isn't going to be enough data stored yet, but we'll forge on blindly ahead now. Let's create our state constructor:

```js
function State() constructor {
	update = undefined;
}
```

Looking pretty good so far. Same general setup, call the constructor with new and we'll get a struct with an update member variable. Update will hold the code that we want the state to run each step. But we've got a few problems. Firstly, we have to directly reach into the innards of the state machine and the struct to set these fields properly, i.e.:

```js
sm = new StateMachine();
idle = new State();
idle.update = function() {
	// Idle code here
}
// Altering state directly here feels a little icky
sm.state = idle;
```

While this works, it's not pretty and it's not very future-proof (what if we want to change the name of the member variable `update` to some other name, like `run`? We'd have to go through our code and find every state we've created and manually edit it to the new name). Much better to create some static methods inside the constructors and assign stuff through them (otherwise known as setters, the companion of getters). We'll start with the state machine and add a setter for assigning a state struct to the state variable.

```js
function StateMachine() constructor {
	state = undefined;

    // Define a static method that we can pass the state struct to, and it does the updating. If we need to change anything about
    // how we assign a state in the future, then we only need to change this one bottleneck here, not every place in the code we change state
	static AddState = function(_state) {
		state = _state;
		return self;
	}
}
```

By returning self from a method, we can chain the calls together, like `.AddState(_state1).AddState(_state2).AddState(_state3);`. This is usually known as a "fluent" interface.
{: .note}

And now we do roughly the same thing for the state.

```js
function State() constructor {
	update = undefined;

    // Another static method that allows us to provide a function the state will run every step as an argument and assign it to update.
	static SetUpdate = function(_update_func) {
		update = _update_func;
		return self;
	}
}
```

Ok, now we can run the methods when adding code to the state and the state to the state machine:

```js
sm = new StateMachine();

// Now we use the new setters instead of direct assignments
idle = new State().SetUpdate(function() { /* Idle code */ });
sm.AddState(idle);
```

That's a bit better. But wait? How are we even going to run the states `update` code? Let's add a method to the state machine to handle that:

```js
function StateMachine() constructor {
	state = undefined;
	static AddState = function(_state) {
		state = _state;
		return self;
	}

    //We'll run this from the state machine, and it'll make run the update function we stored in the state (whichever state is currently assigned to the state variable).
	static Update = function() {
		// First we check to see if our state variable has been set to a struct printed out by a State constructor, using is_instanceof(), so we can be sure it has the update function
		if (is_instanceof(state, State)) {
            // Then we make sure it's actually a function, since update starts off as undefined and perhaps we forgot to assign a function to it
			if (is_method(state.update)) {
                // And finally we run it.
				state.update();
			}
		}
	}
}
```

---

## A Single State Problem

Ok, so now we can create a state, assign it the state machine and then run that state's code in the Step Event by running `sm.Update();`

Excellent. But wait a minute. What happens if we want to change states? Surely we don't want our player stuck in the idle state forever? Ok, so we can't just use a single state variable in the state machine to hold our states. We'll probably also want to name our states, so that there's a string we can read to find out what state we are currently in (maybe print it out for debugging, whatever).

We have two choices here, we can either create an array to hold all of our states, or we can create a struct to hold all our states. An array is easily loopable, and slightly faster access as long as there's not too many entries, but a state lets us directly retrieve a specific state without having to search...

Since we'll be mostly wanting to change to specific states, and will rarely want to loop over them all, I think a struct is best here, since we can use the accessor combined with a state name (for example `states[$ "Idle"]` would retrieve whatever is stored under the `"Idle"` key in the `states` struct, which looks pretty nice to me as a way to grab a specific state), and structs are a little more "human readable" than arrays (less mental gymnastics in the noggin' means more code done in a day).

Finally, we'll want a `ChangeState()` method that lets us swap to another state that has been added to the state machine.

Let's get that setup.

```js
function StateMachine() constructor {
	// We still want our state variable, as it will hold “the current state we are in”
	state = undefined;
	// But we also want a struct to store all of our states in
	states = {};
	
	static Update = function() {
		// First we check to see if our state variable has been set to a struct printed out by a State constructor, using is_instanceof(), so we can be sure it has the update function
		if (is_instanceof(state, State)) {
			if (is_method(state.update)) {
				state.update();
			}
		}
	}

	// And we need to update our AddState so that it adds the new state to the struct, not simply setting state directly to it
	static AddState = function(_state) {
		// As a safety measure, we check to see if a state with the same name already exists in our machine.
		// If we use the accessor to access a struct, and the key doesn't exist in the struct, it returns undefined
		// So we check if it's NOT undefined, which means that the key has already been set for the states struct
		// And therefore, we've already previously added a state with that name
		if (!is_undefined(states[$ _state.name])) {
			// We could exit the function here, or overwrite the state, or whatever we decided
			// was valid behaviour. I'll just throw a debug message to console to warn the programmer
			// And then simply overwrite it
			show_debug_message($"You already have a state named {_state.name} in the state machine!");
		}
		states[$ _state.name] = _state;

        // And we check to see if state has already been assigned to a state struct. If it hasn't, then this is the first state being added to the state machine, and we'll use it as the
        // "starting" state. As a side note, this is why setters are often a good idea, if we had just manually assigned state in a ton of places, and then decided to make a change like
        // this, we'd have a major refactor on our hands, but because we created a setter function early on, all we need to do is change that one function.
        if (is_undefined(state)) {
            state = _state;
        }
		return self;
	}
	
	// Now we make the ChangeState method
	static ChangeState = function(_name) {
		// We run the same check as in Update to see if the state exists in the states struct, and
		// if it does, we simply set state to it.
		if (!is_undefined(states[$ _name])) {
			state = states[$ _name];
		}
		// If it doesn't exist, we warn the programmer that they are trying to change to a state that
		// doesn't exist in the states struct.
		else {
			show_debug_message("Trying to change to a non-existent state!");
		}
	}
}
```

And finally, we want to be able to give names to our states, so that we can use the name as the key for the `states` struct. Let's replace our `State` constructor with a version that takes a name as an argument and assigns it to a `name` variable:

```js
// We give the name as a string for the argument of the State() constructor
function State(_name) constructor {
	name = _name;
	update = undefined;
	static SetUpdate = function(_update_func) {
		update = _update_func;
		return self;
	}
}
```

## Multiple State Disorder

With all that done, we can now, finally, setup our state machine and add multiple states. Create an object (`obj_player` or whatever) and drop this in the **Create Event**:

```js
// Set a general move speed
move_speed = 3;

// Create our state machine
sm = new StateMachine();

// Create an idle state
idle_state = new State("idle")
    // Give the idle state the code we want it to run as an anonymous function through our setter
	.SetUpdate(function() {
        // All idle checks for is whether A or D is pressed, if it is, we consider the player moving and switch to the "move" state
		if (keyboard_check(ord("A")) || keyboard_check(ord("D"))) {
			sm.ChangeState("move");
		}
	});

move_state = new State("move")
	.SetUpdate(function() {
		// Simple movement on the x axis
		var _hor = keyboard_check(ord("D")) - keyboard_check(ord("A"));
		x += _hor * move_speed;
		if (_hor == 0) {
			sm.ChangeState("idle");
		}
	});
sm.AddState(idle_state).AddState(move_state);
```

Now we need to actually run the state machine in the **Step Event**:

```js
sm.Update();
```

And finally, we'll add a little bit of drawing so we can actually see the player and the state's name:

```js
draw_set_color(c_green);
draw_circle(x, y, 32, false);
draw_set_color(c_yellow);
draw_set_halign(fa_center);
draw_set_valign(fa_bottom);
draw_text(x, y - 32, sm.state.name);
```

Make sure you have your object added to your room, run your game, and you should be able to run back and forth, and see the state text above its head change as you switch from moving to standing still. And that's that. We've built a rudimentary state machine. There's many, many ways we could extend it. For instance, keeping track of the last state it was in prior to the current one (or even adding a whole state history to help with debugging). Perhaps we want to be able to run code when a state is entered? Or exited? Or maybe we want to be able to queue state changes? The world is your oyster when it comes to ways you can extend state machines to be more useful. At its core though, they all boil down to the same thing: separating code into buckets, and making sure only one bucket of code runs at a single time.

All that being said, you don't need to develop all this stuff because I've already done it for you (yes it's another plug)! [Check out Statement](https://refreshertowel.github.io/docs/statement/) if you're curious to see what a fully featured state machine framework looks like.

Now I think it's time for me to head back into my lair for a bit, but this time, I won't be gone long. There are things brewing and plans afoot that are coming to fruition soon. Be sure to check back regularly to see what they might be (I feel like I have to check my [Pulse](https://refreshertowel.github.io/docs/pulse/) *ahem* after all this excitement).

Catch you on the flipside my peeps.

[Oh, and here's a downloadable project with all the code in it.](https://github.com/RefresherTowel/refreshertowel.github.io/releases/download/tutorial/Tutorial_BasicStateMachine.yymps)
