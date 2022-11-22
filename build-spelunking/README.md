# Understanding The Build
One of the biggest things devs struggle with is figuring out _exactly_ what caused something to happen in a build. The binlog viewer is your best friend here.

## The Binlog Viewer
[The binlog viewer](https://aka.ms/msbuild/binlog) is accessible via install or on the web, I recommend installing it.

## Binlog Viewer Tips & Tricks

#### Enable these features in the binlog viewer ASAP
1. Open the binlog viewer or go to File -> Start Page
1. Check all the boxes. Particularly "Mark search results with a dot in the main tree"

Now when you search for "Program.cs", the main view will show a dot for every instance where Program.cs showed up. This is extremely useful for chasing down items and how they change throughout the build.

#### "Go to Definition" on a Property
Let's say you want to "Go to Definition" on a property or find places where a property could be overwritten. We can take advantage project files and look up a property based on how it must be defined in XML.

Let's use `TargetFramework` as an example:

- Open a binlog
- Go to the `Find in Files` tab
- Search <TargetFramework>

Suddenly you'll see every instance within a build that would set the property `TargetFramework`. This is useful for finding where the value could be overridden as well. Combine this with [Seeing what imports MSBuild sees]() and you can know exactly when a property is set to a particular value.

Let's say you don't know the full property name. It's `BaseOutput<Something>` for example. Just look up `<BaseOutput` and you'll find all definitions for properties that start with "BaseOutput".

#### Replay Your Binlog
The binlog is amazing, but it isn't perfect. Sometimes not all logged information gets stored in the binlog. If you're concerned you may have fallen into this case, you can replay your binlog.

`msbuild msbuild.binlog /flp:v=diag`

This command replays the build log, outputting everything into an `msbuild.log` file. This may show things that were logged that did not show up in the binlog viewer.

## See Exactly What Imports MSBuild Sees
Running `msbuild myproj.csproj /pp:out.xml` will create a single `out.xml` file that combines every project file/import into a single xml file. This can be useful for understanding what gets defined when, and what properties are set where.


# Working Backwards To "Figure It Out"
Everyone's build scenario is unique and complex. Here's a general guide on using a binlog to figure out what items/properties/targets could be modified to solve a problem you might have!

## 1. "A file was copied to X, but not Y"
Referring to: https://gist.github.com/BenVillalobos/c671baa1e32127f4ab582a5abd66b005?permalink_comment_id=4378272#gistcomment-4378272

You often don't have control over the "default build" logic, so working around this generally involves finding that logic and inserting your logic before it. Here's what's happening (at a high level):

1. Some "copy to the output/publish directory" **target** runs
1. That target copies **items** into Y.
1. Your file is missing from that item.

So the solution then, is the following:

1. Find the target that does the copying.
1. Find the item used by that target to copy.
1. Follow its paper trail (see below) to find the safest target to hook into.
1. Create a target that runs before your target from step 3.
1. Include your file into that item.

#### Finding the target that does the copying
Try searching in the "Search Log" tab for a file you _know_ is copied to a specific directory. Something like `bin\MyApp.dll`, or even just searching `publish\` can help you find something. The targets that copy are usually at the bottom of the results. `CopyFilesToOutputDirectory`, `_CopyOutOfDateSourceItemsToOutputDirectory`, `_CopyFilesMarkedCopyLocal` and `_CopyResolvedFilesToPublishAlways` are good bets here.

#### Finding the item that gets copied
Double clicking a target like `_CopyOutOfDateSourceItemsToOutputDirectory` will show you its underlying XML. Below is an abbreviated version of `_CopyOutOfDateSourceItemsToOutputDirectory`. Notice how it copies the `@(_SourceItemsToCopyToOutputDirectory)` item? This item isn't exactly safe to add directly to. Notice it's marked with an underscore, that implies "private msbuild item, don't use it". At this point, you need to "follow the paper trail" to see how _that_ item gets populated.

```xml
  <Target
      Name="_CopyOutOfDateSourceItemsToOutputDirectory"
      Condition=" '@(_SourceItemsToCopyToOutputDirectory)' != '' "
      Inputs="@(_SourceItemsToCopyToOutputDirectory)"
      Outputs="@(_SourceItemsToCopyToOutputDirectory->'$(OutDir)%(TargetPath)')">

    <Copy
        SourceFiles = "@(_SourceItemsToCopyToOutputDirectory)"
        DestinationFiles = "@(_SourceItemsToCopyToOutputDirectory->'$(OutDir)%(TargetPath)')"
        OverwriteReadOnlyFiles="$(OverwriteReadOnlyFiles)"
        Retries="$(CopyRetryCount)"
        RetryDelayMilliseconds="$(CopyRetryDelayMilliseconds)"
        UseHardlinksIfPossible="$(CreateHardLinksForAdditionalFilesIfPossible)"
        UseSymboliclinksIfPossible="$(CreateSymbolicLinksForAdditionalFilesIfPossible)"
            >

      <Output TaskParameter="DestinationFiles" ItemName="FileWrites"/>

    </Copy>

  </Target>
```

### Following the paper trail
Okay, so `_SourceItemsToCopyToOutputDirectory` is the relevant item, but we don't want to add to it because it's private. Let's see how it gets populated. To do this, go to the "Find in Files" tab in the top right of the binlog viewer. From here, search for `<_SourceItemsToCopyToOutputDirectory`. This will find all instances where `_SourceItemsToCopyToOutputDirectory` is defined. Follow the one in `Microsoft.Common.CurrentVersion.targets`, as that is the "main" build logic. 

After searching, we find:

```xml
<!-- Simplified for the sake of this explanation -->
<_SourceItemsToCopyToOutputDirectory Include="@(_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest)"/>
```

The next piece of the puzzle is `_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest`. Now search for `<_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest`, you'll find:

```xml
<!-- Simplified for the sake of this explanation -->
<_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest Include="@(_ThisProjectItemsToCopyToOutputDirectory->'%(FullPath)')" />
```

Repeat the process one more time and you'll find these lines:

```xml
    <!-- Simplified for the sake of this explanation -->
    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(ContentWithTargetPath->'%(FullPath)')"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(ContentWithTargetPath->'%(FullPath)')"/>
    </ItemGroup>

    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(EmbeddedResource->'%(FullPath)')"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(EmbeddedResource->'%(FullPath)')"/>
    </ItemGroup>

    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_CompileItemsToCopyWithTargetPath)"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_CompileItemsToCopyWithTargetPath)"/>
    </ItemGroup>

    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_NoneWithTargetPath->'%(FullPath)')"/>
      <_ThisProjectItemsToCopyToOutputDirectory Include="@(_NoneWithTargetPath->'%(FullPath)')"/>
    </ItemGroup>
```

Each of these can be followed, but all of them bubble up to [the `AssignTargetPaths` target](), and originate from the `None`, `Content`, `Compile`, and `EmbeddedResource` items.

You can do this process with virtually any item. It gets a lot more difficult to follow in more complex builds, where the build logic goes between different files and based on different imports, etc. But this is a good baseline that should help you figure out a lot of workarounds for yourself.

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

**NOTE**: If you included your file into an item without really knowing if that was the safest place to include it, it may have weird effects on your build. Generally, you want to find the earliest possible place that the build picks up your items and works its magic on them. See [Notable Targets](..\notable-targets\README.md) for more details.
