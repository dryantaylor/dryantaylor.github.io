# New year New blog - The basics of Macgyver
So I'm well aware this comes a couple of days late to be 'new year' but it's close enough so that's what I'm gonna title this post. Since this is the first post I want to focus on the fundamentals of how the engine works currently. This post will assume some familiarity with Unity's GameObject System. 

At a glance it uses a Scene -> GameObjects -> Component system where a single Scene holds many GameObjects which also holds many related Components. Each GameObject has a parent Scene, and each Component has a parent GameObject. Through their parents they can interact with other GameObjects and Components. For example the Macgyver GameObject has the [`getComponentsWithProperty()`](https://github.com/dryantaylor/Macgyver/blob/1dcf91efc6b7c93aca703ff105e8680379194b15/MacGyver/GameObject.h#L45) function much akin to Unity's  [`GameObject.GetComponent()`](https://docs.unity3d.com/ScriptReference/GameObject.GetComponent.html) method. However within Components is where the engine really begins to differ.

## Overview of Components in Macgyver
Whilst Unity has you create a new class which inherits from [`MonoBehaviour`](https://docs.unity3d.com/ScriptReference/MonoBehaviour.html). Macgyver instead treats Components as a set of functions and data structures which the Component stores and manages the calling of each frame.  Taking a look at the [Component definition](https://github.com/dryantaylor/Macgyver/blob/1dcf91efc6b7c93aca703ff105e8680379194b15/MacGyver/Component.h), there are some notible points. The first is `Compoenent::update`  is not a method but a field of the class of type `std::function<void(Component*, unsigned int)>`. That's because when creating a new type of Component instead of a new class, we instead create a new `struct` which has the following template:
```cpp
struct ComponentName {
	static void update(Component* self, unsigned int deltaTime);
	static void attachNew(Component* comp /*,optionally include extra arguments*/ )
	//optional method and only used by physics objects
	static void physicsUpdate(Component* self);
}
```
Then when defining the functions in your .cpp file you put your update logic in the update
function, and any physics updates in the physicsUpdate function.
The `attachNew` function is ran when creating new components of that type, and is passed the `Component` object it will be attached to (however each component can only have one attached set of methods to it). In this function you can attach data structures to the Component, set any flags the component should have, such as `RENDERABLE` or `VELOCITY`, each of which comes with certain promises about what functions or data types it will have. Here is also where you set an `update` function, and optionally a `physicsUpdate` function if the flags you set require one. 

To give an example let's examine the Renderable Component:
>Renderable.h
```cpp
struct RenderableData {
	/*
	Creates a RenderableData struct
	
	@param self pointer to the Component this will be attacheed to
	@param path path to the image relatve to the exe directory
	@param width width to display the image at
	@param height height to display the image at
	*/
	RenderableData(Gameobjects::Component* self, std::string path, 
	               int width, int height);
	
	/*
	Destructor for RenderableData, closes the texture if one is present
	*/
	~RenderableData();

	/// Pointer to internal SDL texture
	SDL_Texture* texture;
	/// SDL rect to contain the width and height to render the 
	/// texture at on screen
	SDL_Rect rect;
};
```
>Renderable.cpp
```cpp
void Components::Renderable::AttachNew(Gameobjects::Component* comp, 
     std::string path, int width, int height)
{
	comp->setComponentProperties(RENDERABLE);
	comp->update = Renderable::update;
	comp->addData(
		componentCreateData(RenderableData, comp, path, width, height)
	);
}
```
On the first Line, we're giving the component the `RENDERABLE` flag, this informs the rest of the engine that this component can be rendered. It also tells the rest of the engine it is allowed to assume the presence of a struct which is `RenderableData` in the Component's Data vector. Note that giving a Component a flag without holding to that flag's promises results in undefined behaviour, and I'm *extra* releasing myself from any liability there.

On the Second line we set the update field of the Component to the Renderable's static update function. This function actually does nothing, however all Components require an update function be set, even if it just immediately returns. 

The third line uses the `componentCreateData` variadic macro to create a new `RenderableData` struct on the heap. The first argument of the macro must be the struct type of the data, whilst following arguments are the arguments to give the constructor in the order they are listed in the constructor.

With that we've created a basic Component in Macgyver, so how does accessing this data work, especially from another Component?

## Accessing Component Data in Macgyver
Accessing Component data isn't quite as simple as `object->field` however , it is still fairly simple. To explain it let's start off by looking at accessing data from the same component in an update method.
 
### Accessing data of the same Component in the update function.
For those familiar with python, the `self` in the update function probably somewhat gave it away, and indeed python's class approach is where I got some of the inspiration from. Whilst there is a `Component::getData` method, for finding a given type we have to call the
[`typeid`](https://en.cppreference.com/w/cpp/language/typeid) operator on the type we're after, and manually cast the returned pointer to the correct type ('ll write about why in a future post where I dive into the inner working of the Component System on a technical level which will explain why). So instead it's simpler to use the `componentGetData()` macro for this purpose. Here we pass in a pointer to the component we're looking for data in, and the type we're after. This will give us the data we're after as a pointer to the type we passed in. For example:
```cpp
PlayerControllerData* playerData = componentGetData(self, PlayerControllerData);
```

### Accessing data of a different Component in the update function.
Here there are two different options depending on if we are looking for data held by a Component in the same GameObject, or a different GameObject (but the same Scene). Either way we are searching for Components which make a commitment to having that data in their flags. 

In the case of the same GameObject we use our `self` object to get it's parent GameObject and then call the GameObjects `getComponentsWithProperty` method. This returns a `std::vector<Component*>` of all Components with the given flag set.

When searching the whole Scene we use the `self` Component's `getWorldScene` method and from the Scene object call its `getComponentsInWorldByType` method, which again returns a `std::vector<Component*>` of all Components with the required flag set.

From there we simply use the same `componentGetData` macro from above, but with the returned Component/s instead of self. 
