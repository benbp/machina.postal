# machina.postal

## OH MY GOSH!  WAT?
So - [machina.js](https://github.com/ifandelse/machina.js) is a JavaScript library for building flexible finite state machines, and [postal.js](https://github.com/ifandelse/postal.js) is a JavaScript message bus library.  "machina.postal" is two things:

1. A very unoriginal name.  I was going to go for something like mail.machine, but seriously, *who* wants to include `<script src="mail.machine.js"><script>` in their project?
2. It's a BRIDGE between machina and postal.  Aw, shoot, did the unoriginal name give that away?

## But How?
When you include machina.postal.js in your project, it sets the reference to machina.bus, and hooks into the "newFsm" event that gets fired anytime a new FSM is created.
In the handler for the "newFsm" event, the machina.postal plugin subscribes to postal, using the FSM's namespace as a channel name, and a wildcard "*" as the topic (will match ANY topic on that channel).  From that point on, if anyone publishes a message on the FSM's channel, with a topic that matches the name of a handler, the FSM will call `handle`, routing the message payload to the handler name (assuming one exists under the current state) which matches the message topic.  Voila!  Your application components can now interact with the FSM over the message bus, and not require a direct reference to it.

But wait, there's more.

The machina.postal plugin also publishes any events generated by the FSM to the message bus, using the FSM's namespace as the channel name (concatenated with ".events" - which you can change if you want).  Again, your components don't need a direct reference to the FSM to subscribe to events, instead they can listen over postal.

## Really, there's not much....let's have a looksee:
The plugin consists of an object literal with the following members:

* `channels` - this is a channel cache for postal (so that new ChannelDefinitions aren't getting created/destroyed all the time).  If you create an FSM named "myFsm" and you don't change the `config` options (below), you will see two channels appear in this cache: "myFsm" and "myFsm.events".  The "myFsm" channel is for handlers, and it will have a member which is an array of postal subscription definitions.  The "myFsm.events" channel is for publishing events, and it will have a member named "eventPublisher" which is the callback subscribed to the "*" event on the FSM, relaying the events to postal.js.
* `config` - object literal containing default "suffixes" for the handler and event channels.  The handlerChannelSuffix defaults to an empty string, while the eventChannelSuffix defaults to ".events".  While it's certainly possible to let the FSM publish event messages and listen for handler messages on the same channel, it's recommended to separate them, in case you happen to name an event identical to a handler name (and thus get double triggering, or, God forbid, a message loop).
* `wireHandlersToBus` - set up function which handles subscribing an FSM to postal.js, using the channel name provided.
* `wireEventsToBus` - set up function which handles subscribing to the FSM "event firehose" (the "*" event) and publishing those to postal.js.
* `wireUp` - handler called anytime a newFsm event is raised, which in turn calls `wireHandlersToBus` and `wireEventsToBus`, passing the newly created FSM to each.

## Using It
Just including the plugin wires it into both postal.js and machina.js - just be sure to include it *after* you include the other two (if you're not going AMD).  Here's a contrived snippet of how it could work:

```javascript
// Let's set up a subscription that will write to the console
postal.subscribe("hunger.machine.events", function(data, envelope) {
	console.log(data.msg);
}

var myFsm = new machina.Fsm({
	initialState: "hungry",

	namespace: "hunger.machine",

	states: {
		hungry: {
			_onEnter: function() {
				this.fireEvent("ShoutIt", { msg: "OH MY GOSH, I'm starving" });
			},

			"go.get.food": function() {
				this.transition("eating");
			}
		},

		eating: {
		    _onEnter: function() {
                this.fireEvent("ShoutIt", { msg: "NOM NOM NOM NOM NOM!" });
            },

            "stop.eating.pig": function() {
                this.transition("satisfied");
            }
		},

		satisfied: {
		    _onEnter: function() {
                this.fireEvent("ShoutIt", { msg: "NAP TIME!" });
            }
		}
	}
});
// when the above FSM instantiates, it will transition into "hungry", and our console subscription
// will print out "OH MY GOSH, I'm starving!"

// now that we have our FSM instance, and a subscription listening for events
// we'll get a channel and publish to it:
var channel = postal.channel("hunger.machine");
channel.publish( { datums: "stuff you might want to send to the FSM handler }, { topic: "go.get.food" } );

// When the "eating" state is entered, the _onEnter handler will fire and publish an event which
// our console subscriber will catch and then print: "NOM NOM NOM NOM NOM!"
// Then we can publish
channel.publish( { datums: "other stuff you might want to send to the FSM }, { topic: "stop.eating.pig" } );

// When the "satisfied" state is entered, the _onEnter handler will fire and publish an event which
// our console subscriber will catch and then print: "NAP TIME!"

```