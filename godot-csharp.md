# Godot 4.x C# Reference Guide

## C# Integration with Godot Engine
A practical reference for C# developers using the Godot 4.x library
Link to the docs: https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_basics.html#introduction

# Table of Contents
1. [Logging](#1-logging)
2. [Signals](#2-signals)
3. [Scene Tree](#3-scene-tree)
   - Common Methods
   - Groups
   - Nodes
   - Scenes
1. [Type Conversion and Casting](#4-type-conversion)
2. [Collections](#5-collections)
3. [Directories and Files](#6-directories-files)
4. [Exported Properties](#7-exported-properties)
5. [Global Classes](#8-globalclass)
6. [Extra C# Notes](#9-extra-notes)

## 1-Logging
String interpellation with $ will work.
```csharp 
GD.Print($"The player is moving in {dir}") 
GD.PushWarning("This is a warning."); 
GD.PushError("Something went wrong.");
```
## 2-Signals
> Using the signal tab in Godot to auto generate the signal in the class doesn't seem to work together with vs code, making you to have to write the signal/method name out yourself.

>According to https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html#signals-as-c-events , you will need to manually disconnect (using -=) all the custom signals you connected as C# events (using +=). So, it is safer to use Connect instead, which will disconnect automatically when nodes are freed.
>copied this example from: https://gist.github.com/xxdocobxx/b3ce929bc2152a4dce09852df10a605b
```csharp
// NodeA.cs
[Signal] public delegate void MySignalEventHandler();
[Signal] public delegate void MySignalWithArgumentEventHandler(string myString);

public void MyMethodEmittingSignals(){
    EmitSignal(SignalName.MySignal);
    EmitSignal(SignalName.MySignalWithArgument, "World");
}

public void ConnectMySignal(Action action){
    Connect(SignalName.MySignal, Callable.From(action));
}

public void ConnectMySignalWithArgument(Action<string> action){
    Connect(SignalName.MySignalWithArgument, Callable.From<string>(action));
}

// NodeB.cs
public override void _Ready(){
    NodeA nodeA = GetNode<NodeA>("NodeA");
    nodeA.ConnectMySignal(OnMySignalCallback);
    nodeA.ConnectMySignalWithArgument(OnMySignalWithArgumentCallback);
}

private void OnMySignalCallback(){
}

private void OnMySignalWithArgumentCallback(string myString){
}
```
## 3-Scene-Tree
### 3.1 Common methods
```csharp 
var tree = GetTree(); 
var tree = GetTree().Root;
var currentScene = GetTree().CurrentScene;
GetTree().ReloadCurrentScene();
GetTree().Quit();
GetTree().Paused = bool;

// Create a timer
await ToSignal( GetTree().CreateTimer(2.0), SceneTreeTimer.SignalName.Timeout );
```
### 3.2 Groups
```csharp 
var enemy = GetTree().GetFirstNodeInGroup("Enemies") as Enemy; 
var enemies = GetTree().GetNodesInGroup("Enemies");
// Call a method on every node in a group 
GetTree().CallGroup("Enemies", Node.MethodName.QueueFree);
```
### 3.3 Nodes
Getting nodes
```csharp
var player = GetParent<Player>();
var parent = GetParent();

var child = GetChild<Node>(0);
var player = GetNode<Player>("Player");

// by path (nested is with "/")
var camera = GetNode<Camera2D>("Player/Camera2D");
// by name in tree
Player player = GetNode<Player>("%Player");

// Check if node exists 
if (HasNode("Player")) { 
var player = GetNode<Player>("Player"); 
}
```
Adding/removing nodes to the tree
```csharp
Node enemy = new Enemy();
AddChild(enemy);
// remove
RemoveChild(enemy);
// delete
enemy.QueueFree();

// check if still active
if (IsInsideTree()){
	GD.Print("Node is active");
}
```
### 3.4 Scenes
Load a scene
```csharp
// Load
PackedScene scene = GD.Load<PackedScene>("res://Scenes/Enemy.tscn");

// instantiate
Enemy enemy = scene.Instantiate<Enemy>();
AddChild(enemy);

// change the scene
GetTree().ChangeSceneToFile("res://Scenes/MainMenu.tscn");

// with PackedScene
GetTree().ChangeSceneToPacked(scene);
```
## 4-Type-conversion
> `GetNode<T>()` casts the node before returning it. It will throw an `InvalidCastException` if the node cannot be cast to the desired type.
```csharp
// with the as operator
Sprite2D mySprite = GetNode("MySprite") as Sprite2D;
// Only call SetFrame() if mySprite is not null
mySprite?.SetFrame(0);

// generic methods
Sprite2D mySprite = GetNode<Sprite2D>("MySprite");
mySprite.SetFrame(0);

// is operator
if (GetNode("MySprite") is Sprite2D){
    // Yup, it's a Sprite2D!
}
```

Changing code depending on the platform: https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_features.html#full-list-of-defines
```csharp
public override void _Ready()
{
	#if (GODOT_MOBILE || GODOT_WEB)
			// Use simple objects when running on less powerful systems.
			SpawnSimpleObjects();
	#else
			SpawnComplexObjects();
	#endif
}
```
## 5-Collections
Godot collections are a wrapper over the collections of c#, making them usually a bit slower, but they are optimized if you want to use them with Godot properties.
Link to the docs: https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_collections.html on what to use when.
Rule of thumb:
- Use **`Godot.Collections.Array<T>`** when interacting with Godot APIs, exported properties, Variants, or GDScript.
- Use **`List<T>`** for most pure C# game logic.
- Use **`T[]`** when you need a fixed-size collection.
```csharp
var godotArr = new Godot.Collections.Array<string>();
var dotnetArr = new string[20]; // fixed size
var dotnetList = new List<string>();
dotnetList.Add("hi"); // c# uses Add (list)
godotArr.Append("!"); // godot uses append

// Godot Array -> C# Array
var godotArray = new Godot.Collections.Array<string>{
	"A",
	"B",
	"C"
};

string[] csharpArray = godotArray.ToArray();
```
### Dictionaries
#### Godot
> for multiple types you can use Variant (Godot) or object(c#)
```csharp
var stats = new Godot.Collections.Dictionary<string, int>();

stats["Health"] = 100;
stats["Mana"] = 50;
stats["Level"] = 1;

GD.Print($"Health: {stats["Health"]}");
GD.Print($"Level: {stats["Level"]}");

// Check if a key exists
if (stats.ContainsKey("Mana")){
	GD.Print($"Mana: {stats["Mana"]}");
}

// Loop through all entries
foreach (var pair in stats){
	GD.Print($"{pair.Key}: {pair.Value}");
}
```
#### C#
```csharp
using System.Collections.Generic;
using Godot;

Dictionary<string, int> stats = new();

stats["Health"] = 100;
stats["Mana"] = 50;
stats["Level"] = 1;
GD.Print(stats["Health"]);

if (stats.TryGetValue("Mana", out int mana)){
	GD.Print($"Mana: {mana}");
}
foreach (KeyValuePair<string, int> pair in stats){
	GD.Print($"{pair.Key}: {pair.Value}");
}
```
## 6-Directories-files
> `DirAccess` ¿ create/delete/list directories and files
> `FileAccess` ¿ read/write files
>Special paths:
>	>  - `res://` ¿ your project files (read-only after export)
>	>  - `user://` ¿ user save data (read/write, recommended for saves)

```csharp
// create direcotry (dir)
Error result = DirAccess.MakeDirAbsolute("user://Saves");
if (DirAccess.DirExistsAbsolute("user://Saves")){}

// nested dir creation
DirAccess.MakeDirRecursiveAbsolute(
    "user://Data/Players/Profiles"
);

// remove file or dir
DirAccess dir = DirAccess.Open("user://Saves");
if (dir != null){
    dir.Remove("save1.txt");
}
// rename
dir.Rename(original, newFile)

// listing folders & files
DirAccess dir = DirAccess.Open("user://Saves");
if (dir != null){
    dir.ListDirBegin();
    string fileName;
    while ((fileName = dir.GetNext()) != ""){
        GD.Print(fileName);
    }
    dir.ListDirEnd();
}

// write to file
using FileAccess file = FileAccess.Open("user://Saves/save1.txt",FileAccess.ModeFlags.Write);
file.StoreString("Player Level: 10");
```
## 7-Exported-properties
https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_exports.html
In Godot, class members can be exported. This means their value gets saved along with the resource (such as the [scene](https://docs.godotengine.org/en/stable/classes/class_packedscene.html#class-packedscene)) they're attached to. They will also be available for editing in the property editor. Exporting is done by using the `[Export]` attribute.
> Works for both properties as fields
```csharp
using Godot;

public partial class ExportExample : Node3D{
	[Export]
	public int Number { get; set; } = 5;
	[Export] public string Name {get; set;} = "Timotheee"

	// some properties are too complex for the analyzer to understand.
	// For example, the following property attempts to use math to display the default value as `5` in the property editor, but it doesn't work:
	[Export] 
	public int NumberWithBackingField
	{
		get => _number + 3;
		set => _number = value - 3;
	}
	private int _number = 2;
	// The analyzer doesn't understand this code and falls back to the default value for `int`, `0`.
	// However, when running the scene or inspecting a node with an attached tool script, `_number` will be `2`, and `NumberWithBackingField` will return `5`. 
	//This difference may cause confusing behavior. To avoid this, don't use complex properties.

// Any type of `Resource` or `Node` can be exported. The property editor shows a user-friendly assignment dialog for these types.
[Export]
public PackedScene PackedScene { get; set; }

[Export]
public RigidBody2D RigidBody2D { get; set; }

// Export grouping
[ExportGroup("My Properties")]
[Export]
public int Number { get; set; } = 3;
// Nested export grouping
[ExportSubgroup("Extra Properties")]
[Export]
public string Text { get; set; } = "";
[Export]
public bool Flag { get; set; } = false;

// Hints (for ex: file
[Export(PropertyHint.File)]
public string GameFile { get; set; }
// String as a path to a file, custom filter provided as hint
[Export(PropertyHint.File, "*.txt,")]
public string GameFile { get; set; }

// ## Limiting editor input ranges[]
[Export(PropertyHint.Range, "0,20,")]
public int Number { get; set; }

// Colors
// Regular color given as red-green-blue-alpha value.
[Export]
public Color Color { get; set; }
// No alpha
[Export(PropertyHint.ColorNoAlpha)]
public Color Color { get; set; }

// Resources
[Export]
public Resource Resource { get; set; }

// enum
public enum MyEnum{
    Thing1,
    Thing2,
    AnotherThing = -1,
}

[Export]
public MyEnum MyEnumCurrent { get; set; }



// and so on, you can almost export anything.
```

### Exporting inspector buttons with `[ExportToolButton]`
https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_exports.html#exporting-inspector-buttons-with-exporttoolbutton
If you want to create a clickable button in the inspector, you can use the [ExportToolButton] attribute. This exports a Callable property or field as a clickable button. Since this runs in the editor, usage of the [Tool] attribute is required. When the button is pressed, the callable is called:
```csharp
[Tool]
public partial class MyNode : Node{
    [ExportToolButton("Click me!")]
    public Callable ClickMeButton => Callable.From(ClickMe);

    public void ClickMe(){
        GD.Print("Hello world!");
    }
}
```
## 8-GlobalClass
Global classes (also known as named scripts) are types registered in Godot's editor so they can be used more conveniently. [In GDScript](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#doc-gdscript-basics-class-name), this is achieved using the `class_name` keyword at the top of a script. This page describes how to achieve the same effect in C#.

```csharp
using Godot;

[GlobalClass]
public partial class Stats : RefCounted{
	[Export] public int Damage { get; set; }
	[Export] public int Health { get; set; }
}
```
Then it will show up in the editor. 

You can also give it a customer logo with:
`[GlobalClass, Icon("res://Stats/StatsIcon.svg")]`
### Some warnings
> The Godot editor will hide these custom classes with names that begin with the prefix "Editor" in the "Create New Node" or "Create New Scene" dialog windows.
> The file name must match the class name in **case-sensitive** fashion. For example, a global class named "MyNode" must have a file name of `MyNode.cs`, not `myNode.cs`.
## 9-Extra-notes
- Currently, projects written in C# cannot be exported to the web platform. To use C# on that platform, consider Godot 3 -instead.
- In Godot's **Editor ¿ Editor Settings** menu: And then set up an external editor **Dotnet** -> **Editor** -> **External Editor**
- You need to (re)build the project assemblies whenever you want to see new exported variables or signals in the editor. This build can be manually triggered by clicking the **Build** button in the top right corner of the editor.
- here are some methods such as `Get()`/`Set()`, `Call()`/`CallDeferred()` and signal connection method `Connect()` that rely on Godot's `snake_case` API naming conventions. So when using e.g. `CallDeferred("AddChild")`, `AddChild` will not work because the API is expecting the original `snake_case` version `add_child`. However, you can use any custom properties or methods without this limitation. Prefer using the exposed `StringName` in the `PropertyName`, `MethodName` and `SignalName` to avoid extra `StringName` allocations and worrying about snake_case naming.
- [NuGet](https://www.nuget.org/) packages can be installed and used with Godot, as with any C# project. Many IDEs are able to add packages directly. They can also be added manually by adding the package reference in the `.csproj` file located in the project root:
