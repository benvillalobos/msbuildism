# What Is A Packaging Project?
__"The project that packages your output into a *.nupkg"__

This is about creating **a separate project** that wrangles together multiple build outputs all in one place.

## To Recreate Yourself
`mkdir the-packaging-project`

`cd the-packaging-project`

`dotnet new console -o ConsoleApp`

`dotnet new classlib -o ClassLib`

`dotnet add ConsoleApp reference ClassLib`

`mkdir packaging`