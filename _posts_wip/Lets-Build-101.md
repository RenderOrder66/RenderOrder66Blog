---
layout: post
title: "Let's Build RTS Basics"
subtitle: "101 An Introduction"
date: 2020-03-24 19:40:00 +1000
background: '/imgs/blogs/lets_build/101/background.png'
author: 'Anthony Reed'
---

# Let's Build 101
## RTS Basics

This is the first in a series of tutorials where we will design and build the basic systems and mechanics of a RTS style game. The real time strategy genre is one of my personal favorites, so it felt a good place to start with a whole tutorial series. But the systems we will build over this tutorial are not just strictly related to the rts genre, when you break them down to the bare basics you will find similar mechanics in genres like World Building, ARPG and Turn Based RPG to name a few.

A few things before we start on this adventure, what will be learning about? Well the various systems make up the general rts genre, Unity engine and software design patterns and architecture, with a focus on S.O.L.I.D principles. A more concise explanation on S.O.L.I.D will come at a later date, but we will go over the importance of following design principles for your projects whether personal or professional, the pros and cons to this design principle and how they apply to games. And as you might have guessed we will be building these systems in Unitys standard mono behaviour OOP way, but may visit on a DOTS hybrid approach when it comes to the performance tutorials.

We will start by setting up the project, you can either clone the repo https://github.com/RenderOrder66/RtsBasics.git or follow along. I have created this in Unity version **2019.3.6f1** as a hdrp project, for the ground I have just added a plane for now, and set up this character prefab made out of a capsule and a cube for 'eyes'. It's always good to have a point of reference for the front of the character. We will assign the navmeshagent component to our character and bake the navmesh surface on our ground. For now this is all we need so let's add a script to our scene to control our character to a target destination.

    public class MousePointToDestination : MonoBehaviour
    {
        [SerializeField] NavMeshAgent agent;
        Camera mainCamera;

        void Start()
        {
            mainCamera = Camera.main;
        }

        void Update()
        {
            if (Input.GetMouseButtonDown(0))
            {
                if(Physics.Raycast(mainCamera.ScreenPointToRay(Input.mousePosition), out var hit, 100))
                {
                    agent.destination = hit.point;
                }
            }   
        }
    }


This is a pretty standard looking script that you would find scattered all across the Interwebs if you searched nav mesh agent destination. It’s pretty simple and does the job of moving the agent to the mouse click destination, but consider this; you require the destination point to be pre calculated by some arbitrary means, well you will need to come back into this class and modify it. And consider this next scenario; you want to change the input to unity’s new input system, this means again modifying this class, and now look at how many scripts you are writing for even the most simplest of games. So how do we fix it so this class becomes more maintainable and open to change, well we’ll start by breaking down what this class actually does or more to the point what is happening in the update method.
We wait for the left mouse down signal
We cast a ray from the camera to the screen point of the mouse cursor
We check if that ray cast hits something.
And lastly we pass the position to the agent.
And from this we can see that the function has already broken the **S**: Single responsibility principle in S.O.L.I.D, and we already know it breaks the **O**: open closed principle by the earlier examples. And secondly this class is coupled to Unity's static Input and Physics implementations, and is dependent on nav mesh agent and camera.

So let’s first separate concerns for the input and physics implementations by abstracting them away from any of our objects that require them. This can be considered a con to most developers that are new to this way of thinking as it requires us to essentially implement what has already been provided to us, but from any of the following architectural design methodologies that will be outlined in this series were to be followed I would follow this one. In a nutshell abstracting the implementation away from any concerns makes your code base much more open to enhancements and upgrades, as we will find out in this series I’m currently using the old Input api’s to start off with even know Unity’s new input system is nearing production ready, we will eventually swap out the old for the new with (hopefully) zero to little changes to our dependents.

### Enter Abstrations

    public interface IMouseUserInput
    {
        Vector3 MousePosition();
        bool SelectionButtonUp();
    }

For now we will just define an interface to provide us with what we require for the above script though this will grow over time. As you can see it’s very simple, all we require is a vector3 for current mouse position and whether or not the mouse is down. You may also notice that the mouse position is a method not a read only property, either or does not affect how this works but it comes down to how for the most part unity built their framework and properties aren't really promodent within their api’s and since the inspector cannot serialize properties without some work or plugins there would be little benefit to this. 

    public interface IPhysicsControl
    {
        bool RayCastHit(Ray ray, out RaycastHit info, float maxDistance);
    }

And now we will abstract away our physics implementation, and for now we will just use the parameters that the unity method required.

    public class PhysicsControl : MonoBehaviour, IPhysicsControl
    {
        public bool RayCastHit(Ray ray, out RaycastHit info, float maxDistance) =>
            Physics.Raycast(ray, out info, maxDistance);
    }

    public class MouseInputController : MonoBehaviour, IMouseUserInput
    {
        public bool SelectionButtonUp() => Input.GetMouseButtonUp(1);
        public Vector3 MousePosition() => Input.mousePosition;
    }

Now let’s make our concrete implementations, and we will do the same for the camera and nav mesh agent. That’s our coupling concerns dealt with now let’s move onto breaking up each responsibility into their own classes, and a good place to start is the input controller as it was the first thing in our list of functional steps from before. For now we will implement a very basic controller that just listens for the `SelectionButtonUp()` and then in turn triggers a unity event where we will be listening for to provide the agent the destination.

But due to unity’s inability to serialize an event that takes a generic argument we need to create the event ourselves and decorate the class with the `Serializable` attribute.

    [Serializable]
    public class VectorEvent : UnityEvent<Vector3> { }

Now we can see the event in the inspector.

So we will make that bare minimum of the input controller for now, but we will come back later to refactor this as the game progresses.


    public class GameInputController : MonoBehaviour
    {
        public VectorEvent onPress;
        IMouseUserInput mouseInput;

        void Awake()
        {
            var components = GetComponents<MonoBehaviour>();
            mouseInput = (IMouseUserInput)components.FirstOrDefault(x => x is IMouseUserInput);
        }

        void Update()
        {
            if (!mouseInput.SelectionButtonUp()) return;
            onPress?.Invoke(mouseInput.MousePosition());
        }
    }

Now all this script does is waits for some kind of signal from the mouse input interface and triggers the appropriate event which in our case is the `onPress`. We are going to clean this up later but just to get things working we will pass the mouse position through the event, this isn’t a good idea to pass the position out from the controller because what if later on we add the ability to attack or gather resources, or even non character related actions such as set a deployment point when a building creates new units. These kinds of actions should be handled by a factory like mechanism. Another note with this controller script you may of noticed is we are now starting the practice of the **D** in S.O.L.I.D principles *dependency injection / dependency inversion* we are supplying the abstraction of `IMouseUserInput` through the start method by finding the type of `IMouseUserInput` in a list of components. This is called property injection, which is not an overly common place in application development which mostly uses constructor injection, but that’s not possible through `MonoBehaviour` as unity handles the initialization of  `MonoBehaviour` through the `AddComponent<T>` method.

    public class MousePointToDestination : MonoBehaviour
    {
        [SerializeField] NavMeshAgent agent;
        IMainCamera mainCamera;
        IPhysicsControl physicsControl;
        void Awake()
        {
    	    var components = GetComponents<MonoBehaviour>();
        	mainCamera = (IMainCamera)components.FirstOrDefault(x => x is IMainCamera);
            physicsControl = (IPhysicsControl)components.FirstOrDefault(x => x isIPhysicsControl);
        }

    	public  void SetAgentDestination(Vector3 destination)
        {
    	    if(!physicsControl.RayCastHit(mainCamera.ScreenPointToRay(destination), out var hit, 100)) return;

            agent.destination = hit.point;
        }
    }

Now in the inspector we will add the `MousePointToDestination.SetAgentDestination(Vector3 destination)` to the `GameInputController.onPress` event. Now this script is fully decoupled from any game controllers \ inputs and is now loosely decoupled from the camera and physics implementations, which means our script needs the two dependencies to work, but does not care at all how dependencies actually work. 

That’s going to be all for this introduction as it is quite long now but in the coming posts we will just be explaining what is required and not so much detail about software design patterns and architecture, as these topics will be covered in standalone posts going into depths.

### Conclusion

So we have covered an introduction into what we will be building and how we will be building it. We made a small script to set a target destination on a `NavMeshAgent`, determined the shortcomings of this script and attempted to resolve the issues by applying the S.O.L.I.D principles and decoupling the script from dependencies. We left it in an unfinished state for now as we will be expanding on it next post when we look at our first rts mechanic, and one of the most important; *Selection* which we will cover single select, box select and target selection. As well as applying this move player to destination script to the collection of currently selected agents, and how we will manage the delegation of selection events to the actual objects we are clicking on.