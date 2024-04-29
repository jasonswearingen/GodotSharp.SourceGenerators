# GodotSharp.SourceGenerators

C# Source Generators for use with the Godot Game Engine (supports Godot 4 and .NET 8!)
* `SceneTree` class attribute:
  * Generates class property for uniquely named nodes
  * Provides strongly typed access to the scene hierarchy (via `_` operator)
* `GodotOverride` method attribute:
  * Allows use of On*, instead of virtual _* overrides
  * (Requires partial method declaration for use with Godot 4.0)
* `Notify` property attribute:
  * Generates boiler plate code, triggering only when values differ
  * (Automagically triggers nested changes for Resource and Resource[])
* `InputMap` class attribute:
  * Provides strongly typed access to input actions defined in godot.project
* `LayerNames` class attribute:
  * Provide strongly typed access to layer names defined in godot.project
* `CodeComments` class attribute:
  * Provides a nested static class to access property comments from code (useful for in-game tooltips, etc)
* `OnInstantiate` method attribute:
  * Generates a static Instantiate method with matching args that calls attributed method as part of the instantiation process
  * (Also generates a protected constructor to ensure proper initialisation - can be deactivated via attribute)
* `OnImport` method attribute (GD4 only):
  * Generates default plugin overrides and options to make plugin class cleaner (inherit from OnImportEditorPlugin)
* Includes base classes/helpers to create project specific source generators

- Version 2.x supports Godot 4
- Version 1.x supports Godot 3

(See [GodotSharp.BuildingBlocks][1] or local tests for example usage patterns)

[1]: https://github.com/Cat-Lips/GodotSharp.BuildingBlocks

## Table of Contents
- [GodotSharp.SourceGenerators](#godotsharpsourcegenerators)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
  - [Attributes](#attributes)
    - [`SceneTree`](#scenetree)
    - [`GodotOverride`](#godotoverride)
    - [`Notify`](#notify)
    - [`InputMap`](#inputmap)
    - [`LayerNames`](#layernames)
    - [`CodeComments`](#codecomments)
    - [`OnInstantiate`](#oninstantiate)
    - [`OnImport`](#onimport)

## Installation
Install via [NuGet](https://www.nuget.org/packages/GodotSharp.SourceGenerators)

## Attributes

### `SceneTree`
  * Class attribute
  * Generates class property for uniquely named nodes
  * Provides strongly typed access to the scene hierarchy (via `_` operator)
  * Nodes are cached on first retrieval to avoid interop overhead
  * Advanced options available as attribute arguments
    * tscnRelativeToClassPath: (default null) Specify path to tscn relative to current class
    * traverseInstancedScenes: (default false) Include instanced scenes in the generated hierarchy
```cs
// Attach a C# script on the root node of the scene with the same name.
// [SceneTree] will generate the members as the scene hierarchy.
[SceneTree]
public partial class MyScene : Node2D 
{
    public override void _Ready() 
    {
        // You can access the node via '_' object.
        GD.Print(_.Node1.Node11.Node12.Node121);
        GD.Print(_.Node4.Node41.Node412);

        // You can also directly access nodes marked as having a unique name in the editor
        GD.Print(MyNodeWithUniqueName);
        GD.Print(_.Path.To.MyNodeWithUniqueName); // Long equivalent
    }
}
```
### `GodotOverride`
  * Method attribute
  * Allows use of On*, instead of virtual _* overrides
  * (Requires partial method declaration for use with Godot 4.0)
  * Advanced options available as attribute arguments
    * replace: (default false) Skip base call generation (ie, override will replace base)
```cs
public partial class MyNode : Node2D 
{
    [GodotOverride]
    protected virtual void OnReady()
        => GD.Print("Ready");

    [GodotOverride(replace: true)]
    private void OnProcess(double delta)
        => GD.Print("Processing");

    // Requires partial method declaration for use with Godot 4.0
    public override partial void _Ready(); 
    public override partial void _Process(double delta); 
}
```
Generates:
```cs
    public override void _Ready()
    {
        base._Ready();
        OnReady();
    }

    public override _Process(double delta)
        => OnProcess(delta);
```
### `Notify`
  * Property attribute
  * Generates public events Value1Changed & Value1Changing and a private class to manage field and event delivery
  * (Automagically triggers nested changes for Resource and Resource[])
  * Events are triggered only if value is different
```cs
public partial class NotifyTest : Node
{
    [Notify]
    public float Value1
    {
        get => _value1.Get();
        set => _value1.Set(value); // You can also pass onchanged event handler here
    }

    public override void _Ready()
    {
        Value1Changing += () => GD.Print("Value1Changing raised before changing the value");
        Value1Changed += () => GD.Print("Value1Changed raised after changing the value");

        // You can also subscribe to private events if needed 
        //   _value1.Changing += OnValue1Changing;
        //   _value1.Changed += OnValue1Changed;
        // This might be useful... erm... if you need to clear the public listeners for some reason...?

        Value1 = 1; // Raise Value1Changing and Value1Changed
        Value1 = 2; // Raise Value1Changing and Value1Changed
        Value1 = 2; // No event is raised since value is the same
    }
}
```
### `InputMap`
  * Class attribute
  * Provides strongly typed access to input actions defined in godot.project (set via editor)
  * If you want access to built-in actions, see [BuiltinInputActions.cs](https://gist.github.com/qwe321qwe321qwe321/bbf4b135c49372746e45246b364378c4)
```cs
[InputMap]
public static partial class MyInput { }
```
Equivalent (for defined input actions) to:
```cs
// (static optional)
// (string rather than StringName for Godot 3)
// (does not provide access to built-in actions)
partial static class MyInput
{
    public static readonly StringName MoveLeft = "move_left";
    public static readonly StringName MoveRight = "move_right";
    public static readonly StringName MoveUp = "move_up";
    public static readonly StringName MoveDown = "move_down";
}
```
### `LayerNames`
  * Class attribute
  * Provides strongly typed access to layer names defined in godot.project (set via editor)
```cs
[LayerNames]
public static partial class MyLayers { }
```
Equivalent (for defined layers) to:
```cs
// (static optional)
partial static partial class MyLayers
{
    public static class Render2D
    {
        public const int MyLayer1 = 0; // Yes, layers start at 1 in editor, but 0 in code
        public const int MyLayer2 = 1;
        public const int MyLayer7 = 6;
        public const int _11reyaLyM = 10; // Yes, we will append an underscore if required...

        public static class Mask
        {
            public const uint MyLayer1 = 1 << 0;
            public const uint MyLayer2 = 1 << 1;
            public const uint MyLayer7 = 1 << 6;
            public const uint _11reyaLyM = 1 << 10;
        }
    }
    public static class Render3D
    {
        public const int MyLayer1 = 0;

        public static class Mask
        {
            public const uint MyLayer1 = 1 << 0;
        }
    }
    // Also for Physics2D, Physics3D, Navigation2D, Navigation3D, Avoidance, etc...
}
```
### `CodeComments`
  * Class attribute
  * Provides a nested static class to access property comments from code
  * Advanced options available as attribute arguments
    * strip: (default "// ") The characters to remove from the start of each line
```cs
[CodeComments]
public partial class CodeCommentsTest : Node 
{
    // This a comment for Value1
    // [CodeComments] only works with Property
    [Export] public float Value1 { get; set; }

    // Value 2 is a field so no comment
    [Export] public float value2;

    public override void _Ready() 
    {
        GD.Print(GetComment(nameof(Value1))); // output: "This a comment for Value1\n[CodeComments] only works with Property"
        GD.Print(GetComment(nameof(value2))); // output: "" (No output for fields, but could be added if needed)
    }
}
```
### `OnInstantiate`
  * Method attribute
  * Generates a static Instantiate method with matching args that calls attributed method as part of the instantiation process
  * (Also generates a protected constructor to ensure proper initialisation - can be deactivated via attribute)
  * Advanced options available as attribute arguments
    * ctor: (default "protected") Scope of generated constructor (null, "" or "none" to skip)
```cs
// Initialise can be public or protected if required; args also optional
// Currently assumes tscn is in same folder with same name
// Obviously only valid for root nodes
public partial class MyScene : Node
{
    [OnInstantiate]
    private void Initialise(string myArg1, int myArg2)
        => GD.PrintS("Init", myArg1, myArg2);
}
```
Generates (simplified):
```cs
    private static PackedScene __scene__;
    private static PackedScene __Scene__ => __scene__ ??= GD.Load<PackedScene>("res://Path/To/MyScene.tscn");

    public static MyScene Instantiate(string myArg1, int myArg2)
    {
        var scene = __Scene__.Instantiate<MyScene>();
        scene.Initialise(myArg1, myArg2);
        return scene;
    }

    private Test3Arg() {}
```
Usage:
```cs
    // ... in some class
    private void AddSceneToWorld()
    {
        var myScene = MyScene.Instantiate("str", 3);
        MyWorld.AddChild(myScene);
    }
```
### `OnImport`
  * Method attribute (GD4 only)
  * Generates default plugin overrides and options to make plugin class cleaner (inherit from OnImportEditorPlugin)
  * Includes base classes/helpers to create project specific source generators
  * (Not that useful unless writing lots of plugins - might remove in v3)
