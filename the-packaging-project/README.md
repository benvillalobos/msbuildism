# What Is A Packaging Project?
__"The project that packages your output into a *.nupkg"__

This is about creating **a separate project** that wrangles together multiple build outputs all in one place.

### To Recreate This Tutorial To Follow Along
If you're curious about how to do this yourself, I highly recommend following along to better understand what to do.

`mkdir the-packaging-project`

`cd the-packaging-project`

`dotnet new console -o ConsoleApp`

`dotnet new classlib -o ClassLib`

`dotnet add ConsoleApp reference ClassLib`

`mkdir packaging`

# Table of Contents
1. [Creating the Packaging Project](#creating-the-packaging-project)
1. [Creating the NuGet Package](#creating-the-nuget-package)
1. Building Your Projects From The Packaging Project
1. Gather Build Outputs
1. Adding Custom Items
1. Customizing The Folder Structure

# Creating The Packaging Project
1. Create a new folder, call it `packaging` or whatever name you prefer.
1. Create a `*.csproj` that matches the name of your folder (not required, but it is convention).
    NOTE: It is **VERY** important that your project's extension is `csproj`. The build process is affected by which extension you give it.
1. Ensure your project uses the [`Microsoft.Build.NoTargets` SDK](https://github.com/microsoft/MSBuildSdks/blob/main/src/NoTargets/README.md). This SDK is intended for projects that aren't meant to be compiled.

# Creating the NuGet Package
