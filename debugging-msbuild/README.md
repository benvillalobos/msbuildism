# Debugging MSBuild
Debugging MSBuild can be difficult, here's the step by step process I usually take.

0. [Understand which MSBuild you want to debug into](..\the-flavors-of-msbuild\README.md).
1. [Set up Visual Studio for Debugging](#1-set-up-visual-studio-for-debugging).
1. [Set up the MSBuild repo](#2-set-up-the-msbuild-repo) if you're unable to step into MSBuild code for some reason.
1. [Replace MSBuild Binaries](#3-replace-msbuild-binaries) if you want to customize MSBuild.
1. [Reproduce Your Scenario](#4-reproduce-your-scenario).
1. [Break Into the Right Function](#5-break-into-the-right-function).

#### Common Issues
1. [Double check your symbols](#double-check-your-symbols).

#### Etc.
1. [MSBuild & the .NET SDK]()

### 1. Set up Visual Studio for Debugging
- Open VS
- Tools -> Options -> Debugging
    - Uncheck "Enable Just My Code"
    - Check .NET Framework source stepping
    - Uncheck "Require source files to exactly match the original version)
        - This may be necessary if you're using the MSBuild repo's source to step through, and the source doesn't match 1:1.

#### Custom Symbol Paths
You may want to set a custom symbol path to force Visual Studio to use your binaries for source-stepping. To do this, Tools -> Options -> Symbols -> + -> `C:/Path/To/MSBuild`. You may want to point it to your SDK folder when debugging an SDK build. Remember to undo this afterward.

## 2. Set up the MSBuild repo
1. `cd <wherever you place your code>`
1. `git clone https://github.com/dotnet/msbuild`
1. `cd msbuild`
1. `build.cmd`
1. `devenv MSBuild.Dev.slnf`

**Note:** Say you want to make sure your MSBuild source code matches the version that's in the .NET SDK or Visual Studio exactly. Run `msbuild --version` (dev cmd prompt) or `dotnet msbuild --version` to figure out which commit hash to checkout. You should see a message like `MSBuild version 17.4.0+18d5aef85 for .NET Framework`, `18d5aef85` is the hash to checkout.

## 3. Replace MSBuild Binaries
This is for those _very_ complex scenarios where you need to manually modify MSBuild and test builds using those new binaries. I typically do this when logging isn't an option and I need a task to spit out some `Console.WriteLine`s.

1. [Set up the MSBuild Repo](#2-set-up-the-msbuild-repo).
1. Make changes and rerun `build.cmd`
1. run `scripts/Deploy-MSBuild.ps1 -Destination:"C:/Path/To/VS/Or/Sdk` (run as admin)
1. Repro your scenario / debug into msbuild / etc.
1. Don't forget to replace your custom bits with the originals afterward. The `Deploy-MSBuild.ps1` automatically creates a backup folder 

If you're manually replacing binaries, they should exist at: `<msbuild-repo-root>/artifacts/bin/bootstrap/<yourTargetFramework>/MSBuild/`. Use net472 for MSBuild under VS, and net* for .NET SDK.

## 4. Reproduce Your Scenario
You want at least one instance of Visual Studio open, ideally with the `MSBuild.Dev.slnf` project open. If your repro scenario requires VS, you'll have two instances of VS open at the same time.

#### Via Command Line
1. Launch a developer command prompt
1. Set the right environment variable: `set MSBUILDDEBUGONSTART=1`
    - 1 = Show a prompt that displays which debuggers to attach to.
    - 2 = Display a process ID, manually attach to it, then continue the build.
1. Run your scenario, break into the process using the Visual Studio that has the MSBuild source open.

#### Via Visual Studio
1. Launch a developer command prompt
1. Set the right environment variable: `set MSBUILDDEBUGONSTART=1`
    - 1 = Show a prompt that displays which debuggers to attach to.
    - 2 = Display a process ID, manually attach to it, then continue the build.
1. `devenv YourProject.sln`
1. Attach with the other instance of Visual Studio.

## 5. Break into the right function
Once your debugger (I'm assuming Visual Studio) is attached to MSBuild, it should be somewhere in XMake.cs at `Debugger.Launch();`. Now it's time to set breakpoints.

If you have the MSBuild source setup locally, you can manually place breakpoints as needed, otherwise:

1. Debug -> Windows -> Breakpoints
1. New -> Function Breakpoint

It's easiest to **include the fully-qualified name** as your function breakpoint. That means `Namespace.ClassName.FunctionName`. 

#### Debugging MSBuild Tasks
All MSBuild tasks, custom or not, have the same entrypoint, `Execute()`. Hooking into official MSBuild tasks might look like `Microsoft.Build.Tasks.Message.Execute`, where custom tasks have unique namespaces & class names.

Relevant links for custom MSBuild tasks:
- [Tutorial: Creating a custom task.](https://docs.microsoft.com/visualstudio/msbuild/tutorial-custom-task-code-generation) The most recent doc on writing tasks.
- [Task Writing.](https://docs.microsoft.com/en-us/visualstudio/msbuild/task-writing?view=vs-2022) A slightly older doc, but still useful.
- [Creating an inline task.](https://docs.microsoft.com/visualstudio/msbuild/msbuild-roslyncodetaskfactory) How to write a task directly into your project file!
- [Debugging An MSBuild Build.](https://gist.github.com/BenVillalobos/c901534892f3249246ccb03bd75ddf91) For debugging into MSBuild itself.

# Common Issues

### Double Check Your Symbols
When debugging, it helps to know _exactly_ which dll's are loaded to debug into.

To check this, Debug -> Windows -> Modules. Sort by name, and look for `Microsoft.Build.*.dll` or `MSBuild.dll` to see which dll's are loaded and where they are. If it's pointing to the wrong dll, you can always [set custom symbol paths](#custom-symbol-paths) to manually correct the issue.


# MSBuild & the .NET SDK
This is mostly the same as debugging regular MSBuild, but all the relevant binaries are going to be in your SDK directory. Typically this is `C:\Program Files\dotnet\sdk\your-version`.

If you want to debug using a specific version of the SDK, I recommend [downloading and using a standalone SDK](https://dotnet.microsoft.com/download/dotnet) (click on the version you're looking for, download from the "binaries" column). This will prevent you from ruining your own SDK install. Remember to use `scripts/Deploy-MSBuild.ps1` from the MSBuild repo.

Note the .NET SDK does not have [the different flavors of MSBuild](../the-flavors-of-msbuild/README.md).

## Custom SDK + Custom MSBuild
Sometimes you need to make a change in the SDK _and_ msbuild at the same time. This is tricky but doable. In a `Developer Command Prompt`, build the SDK repo and run `eng\dogfood.cmd`. The relevant MSBuild binaries will be under `<repo-root>\artifacts\bin\redist\Debug\dotnet\`.

[Reproduce your scenario](#4-reproduce-your-scenario) from this dev command prompt.