# What Is A Packaging Project?
__"The project that packages your output into a *.nupkg"__

This is about creating **a separate project** that wrangles together multiple build outputs all in one place.

## Expected Knowledge
This article was written with the expectation that you have some familiarity with MSBuild. You should understand the following at minimum:
- Properties
- Items
- Targets

### To Recreate This Tutorial To Follow Along
If you're curious about how to do this yourself, I highly recommend following along to better understand what to do.

`mkdir the-packaging-project`

`cd the-packaging-project`

`dotnet new console -o ConsoleApp`

`dotnet new classlib -o ClassLib`

`dotnet add ConsoleApp reference ClassLib`

`mkdir packaging`

## Table of Contents
1. [Creating the Packaging Project](#creating-the-packaging-project)
1. Building Your Projects From The Packaging Project
1. [Creating the NuGet Package](#creating-the-nuget-package)
1. Gather Build Outputs
1. Adding Custom Items
1. Customizing The Folder Structure

## 1. Creating The Packaging Project
First, let's get our project set up.

1. Create a new folder, call it `packaging` or whatever name you prefer.
1. Create a `*.csproj` that matches the name of your folder (not required, but it is convention).
    NOTE: It is **VERY** important that your project's extension is `csproj`. The build process is affected by which extension you give it.
1. Ensure your project uses the [`Microsoft.Build.NoTargets` SDK](https://github.com/microsoft/MSBuildSdks/blob/main/src/NoTargets/README.md). This SDK is intended for projects that aren't meant to be compiled.
1. Define a `TargetFramework` property for your project. This `TargetFramework` will not affect your other projects. See [this link](https://learn.microsoft.com/dotnet/standard/frameworks#supported-target-frameworks) (under `TFM`) for `TargetFramework` values.

## 2. Building Your Projects
In an ideal world, you build your packaging project and it "just handles everything." That means building your packaging project should cause your other projects to build. We do this through `ProjectReference` items.

1. Create a `ProjectReference` item that references each project you want to build.
```xml
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj">
    </ItemGroup>
```
Create a `ProjectReference` per project you want to build, and that is enough to build them whenever `packaging.csproj` is built.

## 3. Creating the NuGet Package
Now that our packaging project builds all other relevant projects, let's create our NuGet package.

1. Set this one property to have your packaging project pack on every single build.
```
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
```

And that's basically it!...sort of. If you build `packaging.csproj` at this point you'll see the following error:

```
NU5017: Cannot create a package that has no dependencies nor content.
```

And this is technically true. We got our packaging project to build its references and to create a NuGet package, but we haven't gathered anything to pack yet!

## 4. Gather Your Build Outputs
Ultimately, it is specially-marked `Content` items that get added to NuGet packages. The next step is to add the build output into `Content`, so that the build will take care of the rest.

First, we need to place each respective `ProjectReference` output into its own item.

1. Add `OutputItemType` metadata to your `ProjectReference` items like so:
```xml
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" OutputItemType="ConsoleAppOutput" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj" OutputItemType="OtherLibOutput" />
    </ItemGroup>
```
This gathers the output of each `ProjectReference` item **into a new item**. Now, we have to insert this new item into `Content`. Unfortunately (or fortunately), you can do this MANY ways to do this.

I'll cover the two most common ways that, in combination, should give you enough flexibility to get your package built and packed properly.

### Use outputs from ProjectReferences directly when packing
#### Pros
- No extra copies into packaging project's bin folder
#### Cons
- Manually referencing a file from the output of a separate project doesn't look nice.
- Lack of

### Gather Outputs To Package Project's Bin folder, THEN pack
#### Pros
- Easier to reference individual files
#### Cons
- Extra copies / storage taken up during a build. For more complex projects, you may want to avoid this.

If your reference is an application, this should be enough to have your packaging project automatically gather the outputs into its `bin/` folder. A `ClassLib` project would require adding `ReferenceOutputAssembly="true"` metadata to the `ProjectReference`, like so:
```xml
    <ItemGroup>
        <ProjectReference Include="../ClassLib/ClassLib.csproj" />
    </ItemGroup>
```