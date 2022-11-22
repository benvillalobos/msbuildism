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

Using a binlog to find more info on how to solve this can be tricky.

#### Finding the target that does the copying
Try searching in the "Search Log" tab for a file you _know_ is copied to a specific directory. Something like `bin\MyApp.dll`, or even just searching `publish\` can help you find something. The targets that copy are usually at the bottom of the results. `CopyFilesToOutputDirectory`, `_CopyOutOfDateSourceItemsToOutputDirectory`, `_CopyFilesMarkedCopyLocal` and `_CopyResolvedFilesToPublishAlways` are good bets here.

#### Finding the item that gets copied
Double clicking a target like `_CopyFilesMarkedCopyLocal` will show you its underlying XML. Below is an abbreviated version of `_CopyFilesMarkedCopyLocal`. Notice how it copies the `@(ReferenceCopyLocalPaths)` item? **That** is the item you might want to insert a custom file into.

```xml
  <Target
      Name="_CopyFilesMarkedCopyLocal"
      Condition="'@(ReferenceCopyLocalPaths)' != ''"
        >
  ...
  ...
      <Copy
        SourceFiles="@(ReferenceCopyLocalPaths)"
        DestinationFiles="@(ReferenceCopyLocalPaths->'$(OutDir)%(DestinationSubDirectory)%(Filename)%(Extension)')"
        SkipUnchangedFiles="$(SkipCopyUnchangedFiles)"
        OverwriteReadOnlyFiles="$(OverwriteReadOnlyFiles)"
        Retries="$(CopyRetryCount)"
        RetryDelayMilliseconds="$(CopyRetryDelayMilliseconds)"
        UseHardlinksIfPossible="$(CreateHardLinksForCopyLocalIfPossible)"
        UseSymboliclinksIfPossible="$(CreateSymbolicLinksForCopyLocalIfPossible)"
        Condition="'$(UseCommonOutputDirectory)' != 'true'"
            >

      <Output TaskParameter="DestinationFiles" ItemName="FileWritesShareable"/>
      <Output TaskParameter="CopiedFiles" ItemName="ReferencesCopiedInThisBuild"/>
      <Output TaskParameter="WroteAtLeastOneFile" PropertyName="WroteAtLeastOneFile"/>

    </Copy>

...
...
```

#### Inserting Your Own Build Logic
The difficult part about this is finding the exact target to hook into. The _safest_ bet is go run before `AssignTargetPaths`. Try something like this:

```xml
<Target Name="InsertItemsToCopy" BeforeTargets="AssignTargetPaths">
  <ItemGroup>
    <Content Include="MyGeneratedFile.css" />
  </ItemGroup>
</Target>
```

You can include your file into `EmbeddedResource`, `Content`, `None`, or `BaseApplicationManifest` and have them automatically copied/included in the build.

**NOTE**: There may have been a ton of metadata added to these items throughout the build process, so it's important to know how to "follow the paper trail" to see how an item gets modified throughout the build.

## 2. Following the paper trail
Will do when there's enough interest.
