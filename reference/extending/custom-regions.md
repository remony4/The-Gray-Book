# Regions

Regions can be descibed as *node-like building blocks that have a hole inside*: 
They do something specific - this is the part where they are similar to nodes.
But they are somehow "unsure about the details", so they let the end-user step in and ask for those details - this is what makes them regions.

In general we can describe regions as nodes with a *callback mechanism*: A way to call back that small patch inside the region, patched by the end-user of the region.

# Region flavors

VL offers several of those callback mechanisms for region developers. 

* Delegate-based Callbacks
  * short lived (stateless)
    * `Func<>` or `Action<>` based
    * based on a custom delegate type (typically declared in C#)
  * long running, process-like (stateful)
    * based on two delegates (one for creating a state, one for updating the state)
* Regions built with the `CustomRegion` API
  * long running with support for border control points

# Delegate-based Regions
Those regions allow a region designer to feed any data into the inside of the region and request back any other data. 
All you need to do is to 
* `Invoke` a delegate of any type - e.g. `(Vector2, Rectangle) -> Boolean`
  * feed the data into the inside of the region by feeding to data to the `Invoke` call
  * use the returning value somehow meaningful

Now what's left is to actually delegate the task of filling in the details to the user. 
You do this by asking for a specific "delegate implementation":
* Create an Input Pin and connect it to the first pin of the Invoke node 

By that you basically let the "delegate implementation" of the user / the inside of the users' region flow into your algorithm. You now can reason about when to call back that patch...

## Custom delegate types
These allow you to define Regions that come with nicely named Pins.
Give them a try! 
* They enhance the readability of the Region
* They allow for several Outputs

For details see: https://github.com/vvvv/VL-Language/issues/5

## Stateful - delegate-based
The basic idea here is that the region is built in a way that it allows for process nodes to be placed inside.

Regions of that flavor can instanciate the user patch by calling the white parts of the inside of the region and then update this patch by calling the grey parts of the inside of the region.

Often enough those regions only manage one instance of the users' patch, but in theory such a region can also manage the whole lifetime of many instances those patches. For examples see the experimental LifeTimeManagers, like `LiveAndLetDie`. 

### Stateful Get your hands dirty

This workflow still needs some more love.

Search for `UserPatch`. You'll find a helper patch that allowed to declare some stateful region. Take this as an example for now. Note that in your specific case you might need to change the delegate type in order to fit your needs. Copy over that UserPatch and refine it on your side.  

# `CustomRegion` API based
We now offer a way to build regions that have `Input Border Controlpoints` (BCP) and `Output BCP`. You even can patch them.

This is powerful feature as it allows the end user stay in the flow. Getting data into or outof the region suddely is effortless.  

Where Delegate-based Regions can get arbitrarily complex, only asking for a small detail via a delegate, Regions based on this CustomRegion API are typically focussing only on small tweaks.

Here are some of those new regions
* Comment
* Do
* Try
* ManageProcess

More are upcoming in the Fuse Library for defining GPU functions, for GPU If- and GPU loop regions.

## Usage

In order to patch a new region, all you need to do is to 
* define a new `Process` 
* have an input pin typed `ICustomRegion` on `Update`.

When you now instanciate your new region via the node browser in your help patch, you will get a region. No questions asked. *If you are unhappy that you didn't get asked for Node or Region by the node browser: Check the configuration menu of that Region input pin*

What you are telling the VL system is: I want to define a region, but actually let me just patch that. VL System, you just need me to hand over the specific usage of the region by the user. Tell me everything about the application/user side, let me reflect over that, and I'll take care of the rest by just patching the logic of the region. 

So, you now can use that custom region instance and ask for the specifics and how the user actually used your region.

The `ICustomRegion` type:
* `Input`, `Outputs` are describing the input and output BCPs. You can see this as a reflection-like API that allows you to reason about the static nature of how your region got used.
* `InputValues` allows to inspect the values that flow into the regions' input BCPs.
* `SetOutputValues` allows to finally write into the output BCPs. These are the values that will flow downstream of the region.

Up to now we only focused on the outside of the region. Now let's tackle the inside:

* `CreateRegionPatch` allows you to instanciate one instance of the user patch, retieving a `ICustomRegionPatch`, a type that basically only allows you to update that users' region patch. 

You normally only want to manage the lifetime of one instance, but are not limited to that - thus the seperation into the *region* (`ICustomRegion`) and the instanciation of the *inside* of the region (`ICustomRegionPatch`).

If you only want to manage one instance you might want to use the helper node `CustomRegionPatch`, that takes care of
* instanciating on instance in the beginning of the existence of the region application
* calling update of the region patch each frame
* disposing the dsiposable parts of the users' patch (e.g. some of those used process nodes)

Please See `Do [Control]` or `ManageProcess [Primitive]` for examples and more comments.

The `Do [Control]` is escpecially interesting as it is very basic.
The regions purpose is a bit thin on the first glance:
It only 
* takes the incoming values
* feeds them to inside perspective input BCPs
* calls the patch
* takes the values that ended up on the inside of the output BCPs
* and finally makes these values accessible to the outside perspective of the region

So basically it does nothing. It just executes the inside. Why would I use such a region ever? As a user I could have just not used the region, and my nodes would have been executed as well, so what's the point? Well, subtelties only. It helps sometimes to structure your code. It is very very much comparable to a code block in a textual language.

Anyway. You want to create a region that does more? Just patch along!

* Take the input BCP values, do something with them and then feed them to the inside of the region
* Or take whatever the user patched as output BCPs, try to reason about them, do something with them and feed the transformed values to the outside of the region.

Now your imagination is needed. 
But note that there are some constraints for your ideas :(
* For every BCP (input or output) the user expects data to flow. Maybe you can patch some logic that focuses on some particular BCPs of a certain data type and for the rest of the BCPs: you just let the data flow unmodified(?)
* Currently - this restriction might fall at some point - the outside data type and the inside data type of one particular BCP is always the same. Splicers on Loop regions actually have different types on the inside and the outside, so maybe at some point those BCPs get supported as well. 

Please let us know of your needs.

