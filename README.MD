# Project Progression

- Kenechukwu Ozoemene: 100841149

- Group: Solo

- Github repository: [GitHub - Teaching-myself-Godot/ez-curved-lines-2d](https://github.com/Teaching-myself-Godot/ez-curved-lines-2d)

[![Star History Chart](https://api.star-history.com/svg?repos=Teaching-myself-Godot/ez-curved-lines-2d&type=date&legend=top-left)](https://www.star-history.com/#Teaching-myself-Godot/ez-curved-lines-2d&type=date&legend=top-left)

![](gantt.png)

> **_NOTE:_**  The use of other code is fully referenced in the github repository. All documentation was a community effort, and I had majority say in the documentation of the real-time SVG scaling system.

## Group Formation

- My course project is a solo endeavour which started off as a personal project/ contribution to a larger project aiming to solve the same problem I'm tackling, and as a  result, I chose to work solo because my project was already an established tool used in projects.

- Roles/ Responsibilities: All development.

> **_NOTE:_**  Although this is a solo project in the view of this course, by nature of this code being a part of a larger open source codebase, I am bound by changes and restrictions of that codebase. This includes contributions from participants not in this course.

## Interactive Media Scenario Information

- Project information: This project aims to bring the power of real-time scalable vector graphics to average game development envirionments in open source game engines like the Godot game engine.

- Project goals: First, aim to reduce the file sizes of game/ project textures. Second, bring back the clean crisp texturing that was previously available in  the early web during the Macromedia/ Adobe flash era. Third, avoid reimplementing the same performance mistakes present in the flash era. Finally, bring the full benefits of the SVG (Scalable Vector Graphics) stack to game and interactive media developers.

- Project purpose: My aforementioned project is aimed to be a developer focused tool first, and as such, my deliverables will be tailored to that experience.

## Developer Expectations (Target Audience)

- Developers should be able to implement scalable vector graphics in their projects without changing their work flow.

- Developers should be able to ship their projects to all target platforms available in the Godot game engine, this includes: HTML5 (web), Linux, Windows, Android, iOS, MacOS.

- Implementation of scalable graphics should take little to no code, and should all be available in the game engine editor.

- User Interfaces (UI) should also be easy to develop with the new system.

- Fallback raster graphics in case of render failures, should be feasible/ implemented by default.

> **_NOTE:_**  Due to the nature of this codebase being a contribution to a larger project, I will be focusing on the SVG scaling rendering system as that was my main focus during the course of development.

## Scene Screenshots

<img src="https://github.com/Teaching-myself-Godot/ez-curved-lines-2d/raw/master/addons/curved_lines_2d/screenshots/scaling_textures.png" title="" alt="setting autoscaling svg textures" data-align="center">

*Setting scalable image in the Godot 4 inspector*

## Implementations

> **_NOTE:_** Due to the fast moving pace of the project, some of these implementations are now outdated, or had to be changed to meet project criteria.

### Singleton

- I implemented a now *depricated* SVG manager as my singleton. This was initialized by a separate plugin manager singleton that loaded the SVG manager into the engine project settings upon initialization, and removed it from the project on close. The SVG managers job was to track caches of SVG draw requests throughouth the project, compile that list into a structured object, and split the computation of rendering vector graphics on multiple cores on a machine. The secondary job of the SVG manager was to handle thread cleanups related to vector graphic rendering.

- My justification for this implementation was to optimize real-time vector graphics rendering in a future-proof fashion. This would have led to the full utilization of future CPU architectures when parsing the SVG XML, leading to a design that would improve alongside hardware improvements. This unfortunately had to be cut due to CORS (Cross Origin Resource Sharing) having a major impact on how WebAssembly projects can be shared on the web.

```mermaid
classDiagram
    class PluginManager {
        <<Singleton>>
        +initialize()
        +register_singleton(name, instance)
        +unregister_singleton(name)
    }

    class SVGManager {
        <<Singleton>>
        +cache_requests : Dictionary
        +thread_pool : Array
        +add_draw_request(request)
        +compile_cache()
        +render_multithreaded()
        +cleanup_threads()
    }

    class Thread {
        +start()
        +stop()
    }

    PluginManager --> SVGManager : manages lifecycle
    SVGManager --> Thread : utilizes *
```

**Sequence Diagram**

```mermaid
sequenceDiagram
    autonumber
    participant Engine
    participant PluginManager as <<Singleton>> PluginManager
    participant SVGManager as <<Singleton>> SVGManager
    participant Thread as Worker Thread

    Engine->>PluginManager: initialize()
    PluginManager->>SVGManager: create instance
    PluginManager->>Engine: register_singleton("SVGManager")

    SVGManager->>SVGManager: compile_cache()
    SVGManager->>Thread: start() for rendering tasks
    Thread-->>SVGManager: return rendered results

    Engine-->>PluginManager: on exit()
    PluginManager->>SVGManager: cleanup_threads()
    PluginManager->>Engine: unregister_singleton("SVGManager")
```

---

## Command Pattern

- With how "engine native" we wanted the SVG rendering system to feel, I decided to use a node based approach for the `SVGTextureHelper` and `SVGTextureResource` classes.

- The `SVGTextureResource` is an engine native resource type I developed that stores all the SVG data needed to perform all the vector calculations. It also stored additional properties that gave developers more power to optimize larger SVGs such as the rasterized image resolution. This was also the key to file size reduction, in the way that it parsed the provided SVG XML and bypassed Godot's rasterizer.

- The `SVGTextureHelper` took an `SVGTextureResource` as an input parameter and acted as the intermediary between the developer, Godot, and native nodes that take natively supported texture types. This significantly cut development time for users of the plugin, as well as giving them a choice between our solution, rasterized graphics, or a combination of both.

- With the implentation of these classes, and how neatly they hooked into the engine workflow, and interface, it was trivial to link changes made to them to the editor `UndoRedo` interface.

```mermaid
classDiagram
    class ICommand {
        <<Interface>>
        +execute()
        +undo()
    }

    class ApplySVGTextureCommand {
        <<Command>>
        -target_node : Node
        -resource : SVGTextureResource
        +execute()
        +undo()
    }

    class SVGTextureHelper {
        +set_resource(resource : SVGTextureResource)
        +apply_texture(target_node : Node)
        +undo_redo_integration()
    }

    class SVGTextureResource {
        <<Resource>>
        +svg_data : String
        +resolution : Vector2
        +parse_svg(xml : String)
        +get_texture() : Texture
    }

    class UndoRedo {
        <<GodotSystem>>
        +add_do_method()
        +add_undo_method()
        +commit_action()
    }

    SVGTextureHelper --> SVGTextureResource : uses
    ApplySVGTextureCommand ..|> ICommand
    ApplySVGTextureCommand --> SVGTextureResource : acts on
    ApplySVGTextureCommand --> UndoRedo : registered to
    SVGTextureHelper --> ApplySVGTextureCommand : creates commands for
```

**Sequence Diagram**

```mermaid
sequenceDiagram
    autonumber
    participant Developer
    participant SVGTextureHelper
    participant ApplySVGTextureCommand as <<Command>> ApplySVGTextureCommand
    participant SVGTextureResource
    participant UndoRedo as <<GodotSystem>> UndoRedo

    Developer->>SVGTextureHelper: apply_texture(target_node)
    SVGTextureHelper->>SVGTextureResource: get_texture()
    SVGTextureHelper->>ApplySVGTextureCommand: create(resource, target_node)
    SVGTextureHelper->>UndoRedo: add_do_method(ApplySVGTextureCommand.execute)
    SVGTextureHelper->>UndoRedo: add_undo_method(ApplySVGTextureCommand.undo)
    SVGTextureHelper->>UndoRedo: commit_action("Apply SVG Texture")
    UndoRedo-->>ApplySVGTextureCommand: execute()
    ApplySVGTextureCommand->>SVGTextureResource: render()
    ApplySVGTextureCommand-->>Developer: update node with texture
```

---

## Factory Pattern

- As mentioned earlier in the command pattern category, this came natively due to my implementation approach. Rather that needing developers to manually create and save an `SVGTextureResource` when using adding a texture to any object, clicking on the texture section of the `SVGTextureHelper` input field automatically creates a new resource that can be populated via the Godot engine's new quick file picker dialogue.

- Things were mainly implemented this way to save time for the developers using our plugin, and to further legitimize the "engine native" philosophy I briefly touched on earlier.

```mermaid
classDiagram
    class IResourceFactory {
        <<Interface>>
        +create_resource(file_path : String) : Resource
    }

    class SVGResourceFactory {
        <<Factory>>
        +create_resource(file_path : String) : SVGTextureResource
        +register_resource(resource : SVGTextureResource)
    }

    class SVGTextureHelper {
        +on_texture_field_clicked()
        +set_resource(resource : SVGTextureResource)
    }

    class SVGTextureResource {
        <<Product>>
        +svg_data : String
        +resolution : Vector2
    }

    IResourceFactory <|.. SVGResourceFactory
    SVGResourceFactory --> SVGTextureResource : creates
    SVGTextureHelper --> SVGResourceFactory : requests creation
```

**Sequence Diagram**

```mermaid
sequenceDiagram
    autonumber
    participant Developer
    participant SVGTextureHelper
    participant SVGResourceFactory as <<Factory>> SVGResourceFactory
    participant SVGTextureResource as <<Product>> SVGTextureResource
    participant GodotUI as FilePickerDialog

    Developer->>SVGTextureHelper: click texture input field
    SVGTextureHelper->>GodotUI: open file picker
    GodotUI-->>SVGTextureHelper: selected file path
    SVGTextureHelper->>SVGResourceFactory: create_resource(file_path)
    SVGResourceFactory->>SVGTextureResource: instantiate with SVG data
    SVGResourceFactory-->>SVGTextureHelper: return new resource
    SVGTextureHelper->>Developer: set_resource(resource)
    Developer-->>GodotUI: confirms new SVG texture applied
```

---

## Combined Overview – Pattern Integration

The following diagram summarizes the interplay between the **Singleton**, **Command**, and **Factory** patterns in the SVG rendering subsystem.  
It demonstrates how the plugin bootstraps through the `PluginManager` (Singleton), produces reusable resources via the `SVGResourceFactory` (Factory), and ties user actions to the editor workflow using the `ApplySVGTextureCommand` (Command).

```mermaid
classDiagram
    %% --- Stereotypes ---
    class PluginManager {
        <<Singleton>>
        +initialize()
        +register_singleton(name, instance)
        +unregister_singleton(name)
    }

    class SVGManager {
        <<Singleton>>
        +cache_requests : Dictionary
        +compile_cache()
        +render_multithreaded()
        +cleanup_threads()
    }

    class SVGResourceFactory {
        <<Factory>>
        +create_resource(file_path : String) : SVGTextureResource
    }

    class SVGTextureResource {
        <<Product>>
        +svg_data : String
        +resolution : Vector2
        +parse_svg(xml : String)
        +get_texture() : Texture
    }

    class SVGTextureHelper {
        +on_texture_field_clicked()
        +set_resource(resource : SVGTextureResource)
        +apply_texture(target_node : Node)
    }

    class ICommand {
        <<Interface>>
        +execute()
        +undo()
    }

    class ApplySVGTextureCommand {
        <<Command>>
        -target_node : Node
        -resource : SVGTextureResource
        +execute()
        +undo()
    }

    class UndoRedo {
        <<GodotSystem>>
        +add_do_method()
        +add_undo_method()
        +commit_action()
    }

    %% --- Relationships ---
    PluginManager --> SVGManager : registers and manages
    SVGManager --> SVGResourceFactory : provides access to
    SVGResourceFactory --> SVGTextureResource : creates instances
    SVGTextureHelper --> SVGResourceFactory : requests creation
    SVGTextureHelper --> SVGTextureResource : manages
    SVGTextureHelper --> ApplySVGTextureCommand : generates commands
    ApplySVGTextureCommand ..|> ICommand
    ApplySVGTextureCommand --> UndoRedo : registered with
    ApplySVGTextureCommand --> SVGTextureResource : acts on
```

---

**Sequence Overview**

This sequence diagram provides a runtime flow of how the patterns interact across the plugin’s lifecycle — from plugin startup to user texture application.

```mermaid
sequenceDiagram
    autonumber
    participant Engine
    participant PluginManager as <<Singleton>> PluginManager
    participant SVGManager as <<Singleton>> SVGManager
    participant SVGResourceFactory as <<Factory>> SVGResourceFactory
    participant SVGTextureHelper
    participant ApplySVGTextureCommand as <<Command>> ApplySVGTextureCommand
    participant UndoRedo as <<GodotSystem>> UndoRedo

    %% Initialization
    Engine->>PluginManager: initialize()
    PluginManager->>SVGManager: create & register()
    SVGManager-->>Engine: ready()

    %% Developer Action
    Engine->>SVGTextureHelper: open texture field
    SVGTextureHelper->>SVGResourceFactory: create_resource(file_path)
    SVGResourceFactory-->>SVGTextureHelper: new SVGTextureResource
    SVGTextureHelper->>ApplySVGTextureCommand: instantiate(resource, node)
    SVGTextureHelper->>UndoRedo: register do/undo methods
    UndoRedo->>ApplySVGTextureCommand: execute()
    ApplySVGTextureCommand->>SVGManager: render_multithreaded()
    SVGManager-->>ApplySVGTextureCommand: rendered texture result
    ApplySVGTextureCommand-->>Engine: update texture in editor
```

---

## Plugin/DLL

- Considering the fact that this project is a massive plugin itself with an expansive history, starting off as an off-hand plugin made to satisfy the request of Prof. Gavin Ball during a game jam, to something that got forked by another party and turned into a moderately bigger project, which later led to the inspiration and creation of the final project by another party, ultimately leading to the latest and greatest version that I am now contributing to. I'd argue that the plugin is not just best suited for a course project, but has been useful to both my GDW and professional projects, alongside the projects of over 100 individuals/ teams that make use of the Godot game engine.
