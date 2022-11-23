## Understanding The Build
One of the biggest things devs struggle with is figuring out _exactly_ what caused something to happen in a build. The binlog viewer is your best friend here.

# Binlog Viewer Tips & Tricks
[The binlog viewer](https://aka.ms/msbuild/binlog) is _fundamental_ in understanding your build these days. A text log may always be the source of truth, but the binlog viewer has some amazing features built into it.

Passing `/bl` to `msbuild` or `dotnet build` will generate a `msbuild.binlog` containing the results of your build. Open it [in the browser](https://live.msbuildlog.com/), or after [use the Binlog Viewer](https://msbuildlog.com/).

Shoutout to [Kirill Osenkov](https://github.com/KirillOsenkov) for creating such an amazing tool!

**Note:** The binlog viewer logs all environment variables, so providing one out in the open might be a security risk.

**Enable these features in the binlog viewer ASAP**

1. Open the binlog viewer or go to File -> Start Page
2. Check all the boxes. "Mark search results with a dot in the main tree" is a must.

Now, when you search for "Program.cs", the main view will show a dot for every instance where Program.cs showed up. This is extremely useful for chasing down items and how they change throughout the build.

## "Go to Definition"
"Go to definition" can be tricky because items & properties are typically defined based on _other_ properties & items. Not to mention, they're often defined in different files. "Go to definition" quickly becomes "Go to _next_ definition". The perfect example is during the `AssignTargetPaths` target, [where `ContentWithTargetPath` is defined by all `Content` items](#go-to-definition-on-items). The build effectively "ignores" `Content` from there on out.

Let's say you want to "Go to Definition" on a Property, Item, or Target.. Perform searches based on how various build concepts are defined. I'll lay it out in too much detail below.

#### "Go to Definition" on Properties
Note properties can also be search based on their usage: `$(TargetFramework)`.

Using `TargetFramework` as an example:

- Open a binlog
- Go to the `Find in Files` tab
- Search <TargetFramework>

Suddenly you'll see every instance within a build that would set the property `TargetFramework`. Combine this with [Seeing what imports MSBuild sees]() and you can find exactly when a property is first created and where it's overidden.

#### "Go to Definition" on Items
Note you can search `@(Content)` to find every usage of an item.

Remember that items can be defined many times and have many values stored in it. Items sometimes "change identities" and become other items, like `ContentWithTargetPath` being defined based entirely on `Content`.

Items can be searched the same way as Properties _and_ Targets, because an item can be defined in an itemgroup and as an output of a task.

```xml
  <!-- Adding Foo.cs to @(Content) -->
  <ItemGroup>
    <Content Include="Foo.cs"/>
  </ItemGroup>

  <!-- AssignTargetPath takes all @(Content) items, assigns `TargetPath` metadata to them, and outputs them into `ContentWithTargetPath` as part of the standard build process. -->
  <AssignTargetPath Files="@(Content)" RootFolder="$(MSBuildProjectDirectory)">
    <Output TaskParameter="AssignedFiles" ItemName="ContentWithTargetPath" />
  </AssignTargetPath>
```

Based on this, we can search `<Content` to find a definition of `@(Compile)` and `"ContentWithTargetPath"` (quoted) to find it defined as an output item.

#### "Go to Definition" on Target
Searching for quoted string is how I typically search for the definitions of targets: `"Build"`, `"ResolveProjectReferences"`.

#### "Go to Definition" on everything
You can use partial names and look for MSBuild syntax too: `<Base`, `Path/>`, `<Compile`, `[MSBuild]::`. 

#### "Go to Definition" Example
[Following the Paper Trail](#following-the-paper-trail)

## Replay Your Binlog
The binlog is amazing, but it isn't perfect. Sometimes not all logged information gets stored in the binlog. If you're concerned you may have fallen into this case, you can replay your binlog.

`msbuild msbuild.binlog /flp:v=diag`

This command replays the build log, outputting everything into an `msbuild.log` file. This may show things that were logged that did not show up in the binlog viewer.

# See what MSBuild Sees
Running `msbuild myproj.csproj /pp:out.xml` will create a single `out.xml` file that combines every project file/import into a single xml file. This can be useful for understanding what gets defined when, and what properties are set where.

# Working Backwards To "Figure It Out"
Everyone's build scenario is unique and complex. Here's a general guide on using a binlog to figure out what items/properties/targets could be modified to solve a problem you might have!

## 1. "A file was copied to X, but not Y"
Referring to: https://gist.github.com/BenVillalobos/c671baa1e32127f4ab582a5abd66b005?permalink_comment_id=4378272#gistcomment-4378272

You often don't have control over the "default build" logic, so working around this generally involves finding that logic and inserting yours before it. Here's what's happening (at a high level):

1. Some "copy to the output/publish directory" **target** runs
1. That target attempts to copy **items** into directory Y.
2. Your file would have been copied, except when:
  - Your file does not exist (see [Including Generated Files](..\including-generated-files\README.md))
  - Your file does exist, and was not in the item (see below).
  - Your file does exist and was in the item, but was missing some sort of metadata. (See [Breadcrumb Trail](#breadcrumb-trail))
    - My first guesses would be `TargetPath` or `FullPath`.

#### Your file exists, but is not in the item

Your solution is to figure out how to insert your file into the build in such a way that it isn't left behind.

The step by step:

1. Find the target that does the copying.
1. Find the item used by that target to copy.
1. Follow its paper trail (see below) to find the safest target to hook into.
1. Create a target that runs before your target from step 3.
1. Include your file into that item.

#### 1. Finding the target that does the copying
Try searching in the "Search Log" tab for a file you _know_ is copied to a specific directory. Something like `bin\MyApp.dll`, or even just searching `publish\` can help you find something. The targets that copy are usually at the bottom of the results. `CopyFilesToOutputDirectory`, `_CopyOutOfDateSourceItemsToOutputDirectory`, `_CopyFilesMarkedCopyLocal` and `_CopyResolvedFilesToPublishAlways` are good bets here.

#### 2. Finding the item that gets copied
Double clicking a target like `_CopyOutOfDateSourceItemsToOutputDirectory` will show you its underlying XML. Below is an abbreviated version of `_CopyOutOfDateSourceItemsToOutputDirectory`.

```xml
  <!-- Simplified for the sake of this example -->
  <Target
      Name="_CopyOutOfDateSourceItemsToOutputDirectory">

    <Copy
        SourceFiles = "@(_SourceItemsToCopyToOutputDirectory)"
        DestinationFiles = "@(_SourceItemsToCopyToOutputDirectory->'$(OutDir)%(TargetPath)')">
    </Copy>

  </Target>
```

We've just discovered that this target copies all `@(_SourceItemsToCopyToOutputDirectory)` items to where they need to be. This is the item we need to add to. 

Next, we "Go to Definition" on it to figure out the safest place to include our file.

#### 3. Follow the Trail
Notice `_SourceItemsToCopyToOutputDirectory` is marked with an underscore to imply "private". Try to avoid modifying private properties & items unless you _really_ know what you're doing.

The next step is to ["Go to Definition"](#go-to-definition-on-items) by searching `<_SourceItemsToCopyToOutputDirectory` under the "Find in Files" tab in the binlog. Follow the definitions in `Microsoft.Common.CurrentVersion.targets`, as that is the "main" build logic. 

After searching, we find:

```xml
<!-- Simplified for the sake of this explanation -->
<_SourceItemsToCopyToOutputDirectory Include="@(_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest)"/>
```

The next piece of the puzzle is `_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest`. ["Go to Definition"](#go-to-definition-on-items) on `_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest` to find:

```xml
<!-- Simplified for the sake of this explanation -->
<_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest Include="@(_ThisProjectItemsToCopyToOutputDirectory->'%(FullPath)')" />
```

Finally, ["Go to Definition'](#go-to-definition-on-items) on `_ThisProjectItemsToCopyToOutputDirectory`:

```xml
    <!-- HEAVILY simplified for the sake of this explanation -->
    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(ContentWithTargetPath->'%(FullPath)')"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(EmbeddedResource->'%(FullPath)')"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_CompileItemsToCopyWithTargetPath)"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_NoneWithTargetPath->'%(FullPath)')"/>
    </ItemGroup>
```

Here we find that `_ThisProjectItemsToCopyToOutputDIrectory` consists of `ContentWithTargetPath`, `EmbeddedResource`, `_CompileItemsToCopyWithTargetPath`, and `_NoneWithTargetPath`. These are beginning to sound familiar!

You can follow these yourself, they all bubble up to [the `AssignTargetPaths` target](..\notable-targets\README.md#assigntargetpaths) and originate from the `None`, `Content`, `Compile`, and `EmbeddedResource` items.

#### Inserting Your Own Build Logic
The difficult part about this is finding the exact target to hook into. The _safest_ bet is go [run before `AssignTargetPaths`](..\notable-targets\README.md#assigntargetpaths). Try something like this:

```xml
<Target Name="InsertItemsToCopy" BeforeTargets="AssignTargetPaths">
  <ItemGroup>
    <Content Include="MyGeneratedFile.css" />
  </ItemGroup>
</Target>
```

You can include your file into `EmbeddedResource`, `Content`, `None`, or `BaseApplicationManifest` and have them automatically copied/included in the build.

**NOTE**: If you included your file into an item without really knowing if that was the safest place to include it, it may have weird effects on your build. Generally, you want to find the earliest possible place that the build picks up your items and works its magic on them. See [Notable Targets](..\notable-targets\README.md) for more an idea of where you could hook your build into.


# The Breadcrumb Trail
1. Go to File -> Start Page
When you check the box for "Mark search results with a dot in the main tree," you're getting one of the best features in the tool. It's easiest to show in screenshots.

It's useful for finding where exactly a file got imported:
TODO

And finding what targets & tasks process which items & properties:
TODO