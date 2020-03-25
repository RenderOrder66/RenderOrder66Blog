---
layout: post
title: "Let's build an RTS - Part 1: The Basics"
subtitle: "A 101 introduction to building an RTS"
date: 2020-03-24 19:40:00 +1000
background: '/imgs/blogs/lets_build/101/background.png'
author: 'Anthony Reed'
---

This is the first in a series of tutorials where we will design and build the basic systems and mechanics of a Real Time Strategy (RTS) style game. The RTS genre is one of my personal favorites, so it felt like a good place to start with a whole tutorial series. But the systems we will build over this tutorial are not just strictly related to the RTS genre. When you break them down into their bare basics, you will find similar mechanics in genres like World Building, ARPG and Turn Based RPG to name a few.

A few things before we start on this adventure: what will be learning about? The various systems that make up the general RTS genre, Unity engine and software design and architecture patterns, with a focus on the S.O.L.I.D principles. A more concise explanation on S.O.L.I.D will come at a later date, but we will go over the importance of following design principles for your projects, whether they be personal or professional, the pros and cons to this design principle and how they apply to games. And as you might have guessed we will be building these systems in Unity's standard Mono Behaviour OOP way, but we may visit a DOTS hybrid approach when it comes to the performance tutorials.

We will start by setting up the project, you can either clone the repo https://github.com/RenderOrder66/RtsBasics.git, or follow along. I have created this in Unity version **2019.3.6f1** as a HDRP project. I've added a plane for the ground, set up this character prefab which is a standard capsule and added a cube for 'eyes'. It's always good to have a point of reference for the front of the character. We will assign the NavMeshAgent component to our character and bake the NavMesh surface on our ground. For now this is all we need so let's add a script to our scene to control our character to a target destination.

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


This is a pretty standard looking script that you would find scattered all across the Interwebs if you searched `nav mesh agent destination`. It’s pretty simple and does the job of moving the agent to the mouse click destination. But consider this: you require the destination point to be pre calculated by some arbitrary means. Well... you would need to come back into this class and modify it. And consider this next scenario: you want to change the input system to implement Unity’s new input system. Once again, this means modifying this class. Now look at how many scripts you are writing for even the most simple of games. So how do we fix it so this class becomes more maintainable and open to change? Well we’ll start by breaking down what this class actually does or more to the point what is happening in the update method.

1. We wait for the left mouse down input
2. We cast a ray from the camera to the point on the screen where the mouse cursor was clicked
3. We check if that ray cast hits something
4. We pass the position to the agent

And from this we can see that the function has already broken the **S**, Single responsibility, principle in S.O.L.I.D. We already know it breaks the **O**, Open Close, principle by the earlier examples. And secondly this class is coupled to Unity's static Input and Physics implementations which  is dependent on the nav mesh agent and the camera.

So let’s first separate concerns for the input and physics implementations by abstracting them away from any of our objects that require them. This can be considered a con to most developers that are new to this way of thinking as it requires us to essentially implement what has already been provided to us, but if I was going to only follow one of the architectural design methodologies outlined in this series, it would be this one. In a nutshell abstracting the implementation away from any concerns makes your code base much more open to enhancements and upgrades, as we will find out in this series. I’m currently using the old Input API to start off with even though Unity’s new input system is nearing production ready. Further into the series we'll swap out the old for the new with (hopefully) zero to little changes to our dependents.

### Enter Abstrations

    public interface IMouseUserInput
    {
        Vector3 MousePosition();
        bool SelectionButtonUp();
    }

For now we will just define an interface to provide us with what we require for the above script though this will grow over time. As you can see it’s very simple, all we require is a Vector3 for current mouse position and whether or not the mouse is down. You may also notice that the mouse position is a method and not a read only property. Either does the trick, but it comes down to how Unity, for the most part, built their Framework. Properties aren't really prominent within their API and since the inspector cannot serialize properties without some work or plugins, there would be little benefit to this. 

    public interface IPhysicsControl
    {
        bool RayCastHit(Ray ray, out RaycastHit info, float maxDistance);
    }

Now we will abstract away our physics implementation, and for now we will just use the parameters that the Unity method requires.

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

Now let’s make our concrete implementations and do the same for the camera and nav mesh agent. That’s our coupling concerns dealt with, now let’s move onto breaking up each responsibility into their own class. A good place to start with this is the input controller as it was the first thing in our list of functional steps from before. For now we will implement a very basic controller that just listens for the `SelectionButtonUp()` and then in turn triggers a Unity event where we will be listening for a position to provide the agent's destination.

But due to Unity’s inability to serialize an event that takes a generic argument, we need to create the event ourselves and decorate the class with the `Serializable` attribute.

    [Serializable]
    public class VectorEvent : UnityEvent<Vector3> { }

Now we can see the event in the inspector.

So we will make that the bare minimum of the input controller for now, but we will come back later to refactor this as the game progresses.


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

Now all this script does is waits for some kind of signal from the mouse input interface and triggers the appropriate event, which in our case is the `onPress` VectorEvent. We are going to clean this up later but just to get things working we will pass the mouse position through the event. This isn’t a good idea to pass the position out from the controller because what if later on we add the ability to attack or gather resources, or even non character related actions such as set a deployment point when a building creates new units. These kinds of actions should be handled by a factory like mechanism. Another note with this controller script: you may of noticed we are now starting the practice of the **D** in S.O.L.I.D principles, *dependency injection / dependency inversion*. We are supplying the abstraction of `IMouseUserInput` through the start method by finding the type of `IMouseUserInput` in a list of components. This is called property injection, which is not an overly common place in application development which mostly uses constructor injection, but that’s not possible through `MonoBehaviour` as Unity handles the initialization of  `MonoBehaviour` through the `AddComponent<T>` method.

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

Now in the inspector we will add the `MousePointToDestination.SetAgentDestination(Vector3 destination)` to the `GameInputController.onPress` event. This script is fully decoupled from any game controllers/inputs and is loosely decoupled from the camera and physics implementations, which means our script needs the two dependencies to work and does not care at all how dependencies are implemented. 

That’s going to be all for this introduction as it is quite long now, but in the coming posts we will just be explaining what is required with not so much detail about software design patterns and architecture, as these topics will be covered in standalone posts going into depths.

### Conclusion

So we have covered an introduction into what we will be building and how we will be building it. We made a small script to set a target destination on a `NavMeshAgent`, determined the shortcomings of this script and attempted to resolve the issues by applying the S.O.L.I.D principles and decoupling the script from dependencies. We left it in an unfinished state for now as we will be expanding on it next post when we look at our first RTS mechanic, and one of the most important; *Selection* which we will cover single select, box select and target selection. As well as applying this move player to destination script to the collection of currently selected agents, and how we will manage the delegation of selection events to the actual objects we are clicking on.