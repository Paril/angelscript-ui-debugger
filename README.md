# angelscript-ui-debugger
A simple, integrated debugger for AngelScript, with ImGui support built-in.

I wrote this framework for a project I'm working on.
I don't have a ton of time to do support or updates, this is just here for anybody who might find any use for it and is provided as-is.
I will be slapping any updates I do directly to this repository - there will likely not be any stable/rolling releases.

![](https://i.imgur.com/60Ep1N1.png)

# What is it?

A set of types and logic to make adding debuggers to AngelScript easy.
In addition, it includes the base for an ImGui-based debugger that functions
similarly to VSCode/MSVC. It's designed to be simple; it likely won't be exactly
what you're looking for if you're looking for something like this, but it will
be close enough that you can tweak it to fit what you need.

# What does it need?

Debugger component:
* AngelScript 2.38.0 minimum ( https://www.angelcode.com/angelscript/wip.php ); this is designed
  on bleeding edge, so it may use features that aren't compatible with 2.37.x and below. YMMV.
* C++17 minimum (this can be worked around, but I wrote this quickly and am targeting C++17 for my uses)
* STL
* fmtlib (this should be able to be substituted for std::format if available)

ImGui UI debugger base:
* ImGui Docking (1.91.9 minimum; might not work with later versions)
* TextEditor (specifically this fork https://github.com/goossens/ImGuiColorTextEdit - this popped up just in time and I love it - give it your support too!)

# How do I implement it? (the debugger)
A full tutorial is not something I plan to write up any time soon, but I will have an example in a live
project some time this month to link to this repository.

The basic jist is as follows:
* subclass `asIDBDebugger` and `asIDBCache`; the latter is not technically necessary but
  you will likely end up doing it anyways.
* implement the abstract functions of the formers' subclass. Exactly what you do during
  Suspend/Resume is going to be dependent on how you're using AngelScript. In my case
  I am using single-threaded execution, so I have `Suspend` create the UI & thread
  for UI tasks and then spin-loop until the UI tells the debugger it is safe to proceed.
* When necessary, create an instance of the debugger subclass. In its initial state, it does nothing and is completely dormant.
  This is by design: I wanted it to be zero-overhead (or as close to it as possible) when the debugger
  has no reason to use precious cycles.
* Whenever you request or create an AS context, check if your debugger is created and if
  HasWork() is true; if so, you should call `HookContext` on the context before `Execute` is called.
  Note that the debugger can only be hooked onto one context at a timne.
* If HasWork() is false, you can safely destroy the debugger. It will remain true as long as
  the debugger has something left to do (it has breakpoints waiting, or it's doing cursor execution).

This on its own is enough to have the debugger 'work'. You only need to make the debugger
if you have work for it to do.

To get things to happen:
* directly call `DebugBreak` on the debugger. This forces the active AngelScript context to immediately break.
  You can then use any sort of interface to interact with the debugger. Undefined behavior will happen
  if you break without an active context.
* add a breakpoint via `ToggleBreakpoint`. You can add a breakpoint onto a function name, or
  a section + line combination. 

# How do I implement it? (the UI)
* subclass `asIDBImGuiFrontend`
* implement the backend functions in the abstract backend methods; these will be called at the correct
  time internally to set up whatever backend you wish to use with the debugger
* the UI will have to be either embedded in a thread or somewhere in your program/game's loop.
  in my case, I don't have direct access to the loop so a thread is used, which complicates
  the code somewhat
* when you wish to show the UI, first call `SetupWindow`; bail if it returns false.
* any time prior to the first call to `Render`, you must call `SetupImGui` once.
* whenever you wish to update the debuggers' UI, call `Render`. The boolean parameter
  indicates whether you want a full render or if you just want a partial update.
  A partial update just keeps the UI alive and guarantees the cache is not dereferenced
  so you can safely swap the cache out while the UI is still active.
* you can ping `SetWindowVisibility` at any time to hide or show the window itself; this is
  not done automatically to prevent annoying flickering. You should do this when you want
  focus to be restored to your application temporarily.
* you can ping `ChangeScript` at any time to ask the UI to refresh the script that is currently
  displayed in the UI.

# How do I use the UI?
* Once broken, the Source tab will automatically show the context for the
  line that it broke on.
* The buttons at the top (or F5, F10, F11 and Shift+F11, same as MSVC) control
  the debuggers' current state.
* The button at the top (or F9) toggle a breakpoint on the currently
  selected line.
* Call stack at the bottom-left can be clicked to change focus on the current
  stack you want to inspect.
* The three first windows on the bottom-right reflect the current state of the stack -
  Parameters, Locals and Temporaries. The first two are self-explanatory; temporaries are
  seemingly allocated by AS for the results of operations.
* Globals shows all globals currently accessible.
* Right-clicking any variable will add it to the Watch window. Right-clicking a variable
  in the watch window will remove it. Note that watch entries only stay for the current
  broken context; changing the debugger state (via Continue, etc) removes all Watch entries.
  This is because there's no guarantee the values in Watch will still be there since it
  renders out the addresses that are fixed at the time of execution.
* Closing the debugger acts as a Continue.

# How do I customize type displays?
* The type display stuff is part of the core of the debugger rather
  than the UI (it's part of the `asIDBCache`)
* In your cache subclass, you can call `evaluators.Register` to register
  a custom `asIDBTypeEvaluator` for any type. This map is completely empty
  by default, and built-in evaluators are used. The built-in evaluators
  (except for null & uninit) are exported in the header for reference, but
  you can also inherit from them if you want to use them.
* Override `Evaluate` to change how the debugger converts an AS object into
  an `asIDBVarValue`, which contains the variables' value, expansion type and
  whether it is disabled or not.
* If your `asIDBVarValue` result is `Entries` or `Children`, override `Expand`
  to handle what is shown when the object is expanded.

The default views should be good for most basic types. It supports
properties & iterating the `foreach` elements. Enums are treated as singular
values unless out-of-band values are specified, in which case it is assumed
to be a bitset and will display the bits and their names (+ the remainder, etc).

Note that the default iteration view will show every iteration value similar to
a multidimensional array; this might look weird for something like `array`, if you're
using the AngelScript CScriptArray, because it supports iterating two values (the value
and the index). For cases like this, the property and foreach iterator functions are
separated, so you can easily make a custom subtype like this:

```cpp
class q2as_asIDBArrayTypeEvaluator : public asIDBObjectTypeEvaluator
{
public:
    virtual void Expand(asIDBCache &cache, asIDBVarAddr id, asIDBVarState &state) const override
    {
        QueryVariableForEach(cache, id, state, 0);
    }
};
```
  
# TODO / coming soon
* remove `ChangeScript` from public API, this shouldn't be necessary
* allow actually scrolling through scripts in the Sections window
* save/load imgui settings. this is disabled atm just so you don't lose functionality by accident
* allow setting the debug action from outside of the debugger (for instance, being able to
  do StepInto before any contexts are loaded so that you automatically step into the first
  context that runs)
* Shift+F10 should step over the entire line. Right now multiple instructions can be
  broken independently, which is useful but probably not always desired.
* figure out the best way to handle namespaces, especially in Globals
* figure out a way to persist Watch properly between call stack entries & execution states
  (although this touches on a probably-not-happening...)

# Probably not happening
* support for multiple modules. right now the code assumes every function/section
  is going to be in the same module. I don't need this for any of my projects, but I welcome contributions.
* support hooking multiple contexts, for certain nested usages
* better error handling; right now the code makes a lot of assumptions about AS stuff
  and doesn't check for errors.
* customizable keyboard shortcuts. this is always a nightmare feature to implement.
  if somebody knows of a good library that can handle it I will embed it.
* hover support for identifiers, etc.
* support for generic expressions in Watch (or at least a way to fetch known entries)
