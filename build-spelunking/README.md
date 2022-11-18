# Understanding The Build
One of the biggest things devs struggle with is figuring out _exactly_ what caused something to happen in a build. The binlog viewer is your best friend here.

## The Binlog Viewer
[The binlog viewer](https://aka.ms/msbuild/binlog) is your best friend when it comes to understanding your build.

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
