# ILLink.Tasks

## Usage

To enable ILLinker set `PublishTrimmed` property to `true` in your project and publish your app as self-contained.

```
dotnet publish -r <rid> -c Release -p:PublishTrimmed=true
```

alternatively you can edit your .csproj file to include 

```xml
  <PropertyGroup>
    <PublishTrimmed>true</PublishTrimmed>
  </PropertyGroup>
```

The output will include only necessary code to run your application. The framework libraries size will be reduced noticeably.

## ILLink Task Properties

### ExtraArgs

Additional [options](illink-options.md) passed to ILLinker.

### OutputDirectory

The directory in which to place processed assemblies.

### ReferenceAssemblyPaths

Assembly files with paths to assemblies needed as references.

### RootAssemblyNames

The names of the assemblies to root. This should contain assembly names without an extension, not file names or
paths.

### RootDescriptorFiles

A list of XML [descriptors](data-formats.md#descriptor-format) files specifying linker roots at a granular level.

## ILLink Task Customization

The linker can be invoked as an MSBuild task, `ILLink`. We recommend not using the task directly, because the SDK has built-in logic that handles computing the right set of reference assemblies as inputs, incremental linking, and similar logic. If you would like to use the [advanced options](illink-options.md), you can invoke the msbuild task directly and pass any extra arguments like this:

```xml
<ILLink AssemblyPaths="@(AssemblyFilesToLink)"
        RootAssemblyNames="@(LinkerRootAssemblies)"
        RootDescriptorFiles="@(LinkerRootDescriptors)"
        OutputDirectory="output"
        ExtraArgs="-t -c link" />
```

## Default Linking Behaviour

### .NET 5.0

By default, the linker will operate in a full linking mode for all framework or 
core managed assemblies. The 3rd party libraries or final app will be analyzed but not linked.
This setting can be altered by setting `_TrimmerDefaultAction` property to different
linker action mode.

### .NET 3.x

By default, the linker will operate in a conservative mode that keeps
all managed assemblies that aren't part of the framework (they are
kept intact, and the linker simply copies them).

Applications or frameworks (including ASP<span />.NET Core and WPF) that use reflection or related dynamic features will often break when trimmed, because the linker does not know about this dynamic behavior, and can not determine in general which framework types will be required for reflection at runtime. To trim such apps, you will need to tell the linker about any types needed by reflection in your code, and in packages or frameworks that you depend on. Be sure to test your apps after trimming.

If your app or its dependencies use reflection, you may need to tell the linker to keep reflection targets explicitly. For example, dependency injection in ASP<span />.NET Core apps will activate
types depending on what is present at runtime, and therefore may fail
if the linker has removed assemblies that would otherwise be
present. Similarly, WPF apps may call into framework code depending on
the features used. If you know beforehand what your app will require
at runtime, you can tell the linker about this in a few ways.

For example, an app may reflect over `System.IO.File`:
```csharp
Type file = System.Type.GetType("System.IO.File,System.IO.FileSystem");
```

To ensure that this works:

- You can include a direct reference to the required type in your code
  somewhere, for example by using `typeof(System.IO.File)`.

- You can tell the linker to explicitly keep an assembly by adding it
  to your csproj (use the assembly name *without* extension):

  ```xml
  <ItemGroup>
    <TrimmerRootAssembly Include="System.IO.FileSystem" />
  </ItemGroup>
  ```

- You can give the linker a more specific list of types or members to
  include using an xml [descriptor](data-formats.md#descriptor-format) file

  `.csproj`:
  ```xml
  <ItemGroup>
    <TrimmerRootDescriptor Include="TrimmerRoots.xml" />
  </ItemGroup>
  ```

  `TrimmerRoots.xml`:
  ```xml
  <linker>
    <assembly fullname="System.IO.FileSystem">
      <type fullname="System.IO.File" />
    </assembly>
  </linker>
  ```

## Caveats

Sometimes an application may include multiple versions of the same
assembly. This may happen when portable apps include platform-specific
managed code, which gets placed in the `runtimes` directory of the
publish output. In such cases, the linker will pick one of the
duplicate assemblies to analyze. This means that dependencies of the
un-analyzed duplicates may not be included in the application, so you
may need to root such dependencies manually.
