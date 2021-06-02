
# Streamline Windows Forms application configuration and bootstrap 

<!--
Provide the primary contacts here. Linking to the GitHub profiles is useful
because it allows tagging folks on GitHub and discover alternative modes of
communication, such as email or Twitter, if the person chooses to disclose that
information.

The bolded roles makes it easier for people to understand who the driver of the
proposal is (PM) and who can be asked for technical questions (Dev). At
Microsoft, these roles happen to match to job titles too, but that's irrelevant.
-->

**Owner** [Igor Velikorossov](https://github.com/RussKie)

# Overview

Historically Windows Forms designer was rendering its design surface based on hardcoded assumptions, such as control default font and dpi settings. However over the past 10-15 years not only multi-monitor configurations became a norm, monitor tech has made significant progress in making 4K, 5K, 8K, etc. monitors widely used by our consumers. The Windows itself has been innovating and tinkering with UI configurations, such default font metrics, which over the years deviated from "Microsoft Sans Serif, 8pt" and now is set to "Segoe UI, 9pt". Incidentally the office team is currently [in search of the next default font](https://www.microsoft.com/en-us/microsoft-365/blog/2021/04/28/beyond-calibri-finding-microsofts-next-default-font/).

The .NET Core/.NET-powered Windows Forms runtime has been (more or less) keeping up with the changes, e.g. by providing new API to [set high dpi mode](https://github.com/dotnet/winforms/pull/396), or by updating the [default font](https://github.com/dotnet/winforms/pull/656). The default font change has unearthed numerous issues in the runtime (e.g. different font-based scaling), which we addressed by providing (yet another) API to set an [application-wide default font](https://github.com/dotnet/winforms/pull/4911). 

However during numerous discussions the team has identified probably the biggest flaw in the current separation of the runtime and the designer, and the overall move from .NET Framework assumptions - the lack of a mechanism to share Windows Forms project-level configuration settings between the runtime and the designer. That is, if the app is configured to run with disabled visual styles, in PerMonitorV2, and have an application-wide default font set to "Arial, 14pt" there is no way to show the designer surface with the same settings to truly provide the WYSIWYG experience.



# Proposal

The purpose of this proposal is to:

1. **streamline a Windows Forms application configuration and bootstrap**, 
2. with the view that this will facilitate the **sharing of the information between the runtime and the designer during the development phase**.<br/>
That is, whenever the designers surface process is started configuration information is read from a known location, and necessary configurations are applied (e.g. run the design surface in PerMonitorV2 mode, or set a form/usercontrol default font to "Arial, 14pt").<br/>
:warning:  **out of scope**, tracked under https://github.com/dotnet/winforms-designer/issues/3192.

**NOTE:** The new functionality is opt-in, i.e. unless a developer makes a conscious decision to use the new configuration and the bootstrap mechanism existing applications will continue to work as-is, and the current developer experience will remain the same.


## Scenarios and User Experience

<!--
Provide examples of how a user would use your feature. Pick typical scenarios
first and more advanced scenarios later.

Ensure to include the "happy path" which covers what you expect will satisfy the
vast majority of your customer's needs. Then, go into more details and allow
covering more advanced scenarios. Well designed features will have a progressive
curve, meaning the effort is proportional to how advanced the scenario is. By
listing easy things first and more advanced scenarios later, you allow your
readers to follow this curve. That makes it easier to judge whether your feature
has the right balance.

Make sure your scenarios are written in such a way that they cover sensible end-
to-end scenarios for the customer. Often, your feature will only cover one
aspect of an end-to-end scenario, but your description should lead up to your
feature and (if it's not the end result) mention what the next steps are. This
allows readers to understand the larger picture and how your feature fits in.

If you design APIs or command line tools, ensure to include some sample code on
how your feature will be invoked. If you design UI, ensure to include some
mock-ups. Do not strive for completeness here -- the goal of this section isn't
to provide a specification but to give readers an impression of your feature and
the look & feel of it. Less is more.
-->

Refer to [Design](#Design) section below.

![projectconfiguration](https://user-images.githubusercontent.com/4403806/119747720-aa0a1b80-bed6-11eb-8247-45b1668d4867.gif)

## Requirements

### Goals

<!--
Provide a bullet point list of aspects that your feature has to satisfy. This
includes functional and non-functional requirements. The goal is to define what
your feature has to deliver to be considered correct.

You should avoid splitting this into various product stages (like MVP, crawl,
walk, run) because that usually indicates that your proposal tries to cover too
much detail. Keep it high-level, but try to paint a picture of what done looks
like. The design section can establish an execution order.
-->

1. The new bootstrap experience (this includes source generators and analyzers) must come inbox with the Windows Desktop SDK, and be available without any additional references or NuGet packages from get go (i.e. `dotnet new winforms && dotnet build`).<br/>Related: [dotnet/designs#181](https://github.com/dotnet/designs/pull/181)
1. The new bootstrap API must only work for Windows applications projects (i.e. `OutputType = WinExe`). These projects have an entry point, where an app is initialised; and they also specify an application manifest, if there is one.

    Property | TFM | Visible
    --|--|--
    `ApplicationVisualStyles` | .NET Framework 2.0+<br/> .NET Core 3.x<br/>.NET 5.0+ |   yes<br/>yes<br/>yes
    `ApplicationUseCompatibleTextRendering` | .NET Framework 2.0+<br/> .NET Core 3.x<br/>.NET 5.0+ | yes<br/>yes<br/>yes
    `ApplicationHighDpiMode` | .NET Framework 2.0+<br/> .NET Core 3.x<br/>.NET 5.0+ | no<br/>yes<br/>yes
    `ApplicationFont*` | .NET Framework 2.0+<br/> .NET Core 3.x/.NET 5.0<br/>.NET 6.0+ | no<br/>no<br/>yes

3. The Windows Forms application template must be updated with the new bootstrap experiece.

### Stretch Goals

1. Update Visual Studio property page for Windows Forms projects.
2. Build the following Roslyn Analyzers functions:

    * Check for invocations of now-redundant `Applicaiton.*` methods invoked by `ApplicationConfiguration.Initialize()`.
    * If a custom app.manifest is specified, parse it, and if dpi-related settings are found - warn the user, and direct to supply the dpi mode via the MSBuild property defined below.
    * (Consider) checking for app.config and dpi-related configurations, if found - warn the user, and direct to supply the dpi mode via the MSBuild property defined below.
    * Check if `Application.SetHighDpiMode()` is invoked with anything other than `HighDpiMode.PerMonitorV2` (see: [dotnet/winforms-designer#3278](https://github.com/dotnet/winforms-designer/issues/3278))


### Non-Goals

<!--
Provide a bullet point list of aspects that your feature does not need to do.
The goal of this section is to cover problems that people might think you're
trying to solve but deliberately would like to scope out. You'll likely add
bullets to this section based on early feedback and reviews where requirements
are brought that you need to scope out.
-->

1. Modify the Windows Forms Designer process to read values either from *.myapp file (VB) or MSBuild properties (C#). This work will be tracked in separately, targeting Dev17.
1. Design/implement top level statements fo Windows Forms applications.
1. Design/implement new host/builder model for Windows Forms applications.
1. Migrate Visual Basic apps off the Application Framework or change the use of *.myapp file.
1. Streamline the bootstrap of Visual Basic apps or use Roslyn source generators for this purpose.
1. Use Roslyn analyzers in Visual Basic scenarios until previous two items are addressed.


:thinking: Strategically it could be benefitial to migrate Visual Basic off the Application Framework and *.myapp file in favour of Roslyn source generators and MSBuild properties. This could also significantly reduce efforts in maintaining Visual Studio property pages for Visual Studio projects (e.g. [dotnet/project-system#7236](https://github.com/dotnet/project-system/issues/7236), [dotnet/project-system#7240](https://github.com/dotnet/project-system/issues/7240), and [dotnet/project-system#7241](https://github.com/dotnet/project-system/issues/7241))



## Stakeholders and Reviewers

<!--
We noticed that even in the cases where we have specs, we sometimes surprise key
stakeholders because we didn't pro-actively involve them in the initial reviews
and early design process.

Please take a moment and add a bullet point list of teams and individuals you
think should be involved in the design process and ensure they are involved
(which might mean being tagged on GitHub issues, invited to meetings, or sent
early drafts).
-->

* Windows Forms runtime/designer: @dotnet/dotnet-winforms @OliaG @DustinCampbell 
* Visual Basic: @KathleenDollard  @KlausLoeffelmann
* Roslyn: @jaredpar @sharwell
* Project System: @drewnoakes 
* SDK: dsplaisted 
* .NET libraries: @ericstj
* General: @terrajobst @Pilchie 

## Design

<!--
This section will likely have various subheadings. The structure is completely
up to you and your engineering team. It doesn't need to be complete; the goal is
to provide enough information so that the engineering team can build the
feature.

If you're building an API, you should include the API surface, for example
assembly names, type names, method signatures etc. If you're building command
line tools, you likely want to list all commands and options. If you're building
UI, you likely want to show the screens and intended flow.

In many cases embedding the information here might not be viable because the
document format isn't text (for instance, because it's an Excel document or in a
PowerPoint deck). Add links here. Ideally, those documents live next to this
document.
-->


### 1. Visual Basic 

Ironically Visual Basic apps are in a good position for the designer integration with their Application Framework functionality, i.e. "*.myapp" file:

![image](https://user-images.githubusercontent.com/4403806/114354200-3d9abd80-9bb1-11eb-9d48-a43f299f82e1.png)

The designer surface process should have no issues reading configuration values from a myapp file. Any work further work at this stage pertaining for streamlining the bootstrap is out of scope of this design proposal.



### 2. C#

```diff
    static class Program
    {
        [STAThread]
        static void Main()
        {
+           ApplicationConfiguration.Initialize();

-           Application.EnableVisualStyles();
-           Application.SetDefaultFont(new Font(....));
-           Application.SetHighDpiMode(HighDpiMode.PerMonitorV2);
-           Application.SetCompatibleTextRenderingDefault(false);

            Application.Run(new MainForm());
       }
    }
```

New MSBuild properties:

```diff
// proj

  <PropertyGroup>
     <ApplicationIcon />
     <ApplicationManifest>app1.manifest</ApplicationManifest>
+    <ApplicationVisualStyles>[true|false, default=true]</ApplicationVisualStyles>
+    <ApplicationUseCompatibleTextRendering>[true|false, default=false]</ApplicationUseCompatibleTextRendering>
+    <ApplicationFont>[equivalent to Font.ToString(), string, default='', i.e. Control.DefaultFont]</ApplicationFontName>
+    <ApplicationFontResx>[name of resource that hold an equivalent to Font.ToString(), string, default='']</ApplicationFontSize>
+    <ApplicationHighDpiMode>[dpi mode, string/HighDpiMode enum value, default=PerMonitorV2]</ApplicationHighDpiMode>
  </PropertyGroup>
```

Existing properties of interest:

```xml
// proj

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <ApplicationManifest>[manifest file, optional, default='']</ApplicationManifest>
  </PropertyGroup>
```

Unlike Visual Basic projects C# projects require a little more work. There are several candidates for storing settings, such as an MSBuild project file (e.g. csproj/props/targets/etc.), an app.config or some external configuration file.

After deliberations and discussions we propose the settings to be stored as MSBuild properties for the following reasons:
- The designer surface process fun in Visual Studio should have no issues reading MSBuild properties.
- Unlike schema-enforced app.config, MSBuild properties are stored in an untyped propertybag.
- app.configs don't appear to be used in .NET apps.
- Whilst developers who build their apps on .NET Framework 4.7+ maybe familiar with [app.config ApplicationConfigurationSection](https://docs.microsoft.com/dotnet/framework/configure-apps/file-schema/winforms/windows-forms-add-configuration-element) and/or [app.config AppContextSwitchOverrides](https://docs.microsoft.com/dotnet/framework/configure-apps/file-schema/runtime/appcontextswitchoverrides-element), we have removed all dependencies on app.config and these quirks in .NET Core 3.0 timeframe.
  > :thought_balloon:  It is also worth noting that quirks were necessary in .NET Framework, which was installed in-place, and didn't provided any kind of side-by-side deployment. In a rare and unfortunate event where we may accidentally break end users experiences in a given release users will be able to install a previous working .NET release and run their application against it until a fix becomes available.
- There may also be an argument that app.config can be changed by end users, thus allowing to alter behaviours of applications without a recompilation, making the app.config the prefered configuration vehicle. But it is important to note that our SDK is built with developers in mind, and not the ability of end users to alter the application behaviours at will. If an app developer feels it is important to allow end users to change the default font or dpi settings via an app.config it is the developers responsibility to facilitate that mechanism to end users.



The runtime portion will leverage Roslyn source generators to read MSBuild configurations, and emit code for the necessary API, e.g. call `Application.SetHighDpiMode(HighDpiMode.PerMonitorV2)` and `Application.SetDefaultFont("Arial", 14f)`.<br/>
:warning: The screenshot is dated, provided for concept demonstration purposes only:
![image](https://user-images.githubusercontent.com/4403806/114354384-8488b300-9bb1-11eb-855a-b58e80953793.png)

We will be able to leverage the power of Roslyn analyzers to warn developers about duplicate/redundant API invocations performed outside the generated code, unnecessary DPI settings, and generally steer them towards the new bootstrap experience.



### 3. Visual Studio

To complete the user experience we can update the Visual Studio property page for Windows Forms projects, and provide an Application Framework-esque (yet arguably more refined) experience to C# developers. It can looks something like this mockup:

![image](https://user-images.githubusercontent.com/4403806/119922957-2e7e9c00-bfb4-11eb-8086-c002c2315c58.png)


Tracked under [dotnet/project-system#7279](https://github.com/dotnet/project-system/issues/7279)



## Q & A

<!--
Features evolve and decisions are being made along the road. Add the question
as a subheading and provide the explanation for the decision below. This way,
you can easily link to specific questions.

When you find yourself having to explain something in a GitHub discussion or in
email, consider to update your proposal and link to your answer instead. This
way, you avoid having to explain the same thing over and over again.
-->

### Font Configuration

#### Default scenario

Initial thinking was to allow configuration of only [`Font.FamilyName`](https://docs.microsoft.com/dotnet/api/system.drawing.font.fontfamily) and [`Font.Size`](https://docs.microsoft.com/dotnet/api/system.drawing.font.size) properties. However these two properties may be insufficient in some use cases, which otherwise would be achievable if `Application.SetDefaultFont(Font)` API was invoked manually.
E.g.:

```cs
Application.SetDefaultFont(new Font(new FontFamily("Calibri"), 11f, FontStyle.Italic | FontStyle.Bold, GraphicsUnit.Point));
```

It would rather be impractical to provide an MSBuild properties for each argument the `Font` constructor takes, and instead the proposal is to configure fonts via a single property `ApplicationFont`. This property will have a format equivalent to the output of `Font.ToString()` API, where the first two tokens required, and all other tokens optional; e.g.:

```
Name=fontName, Size=size[, Units=units[, GDiCharSet=gdiCharSet[, GdiVerticalFont=boolean]]]
```

#### Locale aware font configuration

In addition to this, it is possible that [`Font.FamilyName`](https://docs.microsoft.com/dotnet/api/system.drawing.font.fontfamily) may be locale sensitive, i.e. a different font family (and size) could be used in different regions of the world. To address this use case the proposal is to add another MSBuild property `ApplicationFontResx`, that will contain the name of a localizable resource that contains [`Font.FamilyName`](https://docs.microsoft.com/dotnet/api/system.drawing.font.fontfamily) and [`Font.Size`](https://docs.microsoft.com/dotnet/api/system.drawing.font.size) properties for each locale that requires it.

The order of precedence is as follows:
- `ApplicationFontResx`, if present
- `ApplicationFont`.

#### Error scenarios

* If the value of `ApplicationFont` resource can't  be parsed, this will result in a compilation error.
* If `ApplicationFontResx` contains an invalid resource name, this will result in a compilation error.
* If the value of `ApplicationFontResx` resource can't  be parsed, this will result in a runtime error.



### How to resolve dpi settings?

At this point an astute Windows Forms developer will say that there are currently 3 different ways to configure dpi settings (which may or may not be mutually exclusive or complimentary):
* [app.manifest](https://docs.microsoft.com/windows/win32/sbscs/application-manifests) - Common Control related configuration, dealing with it was always a pain. Very likely that a significant part of Windows Forms developers have never heard of it.
* [app.config ApplicationConfigurationSection](https://docs.microsoft.com/dotnet/framework/configure-apps/file-schema/winforms/windows-forms-add-configuration-element) and [app.config AppContextSwitchOverrides](https://docs.microsoft.com/dotnet/framework/configure-apps/file-schema/runtime/appcontextswitchoverrides-element).
* [`Application.SetHighDpiMode(HighDpiMode)`](https://docs.microsoft.com/dotnet/api/system.windows.forms.application.sethighdpimode).

This proposal introduces a 4th way of configuring dpi, and to make it successful it has to (paraphrasing JRR Tolkien):


> _One API to rule them all, One API to find them,_<br/>
> _One API to bring them all and in the ApplicationConfiguration bind them..._


The benefit the new approach provides is by facilitating the sharing of the information between the runtime and the designer during the development phase, as well as unifying how dpi settings are configured. This benefit is believed to outweigh the need to remove several lines of code from `Main()` method and adding several MSBuild properties.

### Go further?

At this point an astute reader may ask why no go further and offer an API similar to a ASP.<!-- -->NET Core bootstrap? Wouldn't it be nice to write something like:
```cs
[STAThread]
static void Main()
{
    Application
        .Configure(() =>
        {
            // application configuration logic
        })
        .Run(new Form1());
}
```

Indeed, this sounds like a grand idea. However in reality Windows Forms app often have elaborate application bootstrap logic (e.g. [this](https://github.com/gitextensions/gitextensions/blob/master/GitExtensions/Program.cs), [this](https://github.com/luisolivos/Halconet/blob/4ed328a15ba69aab00e92076774977143790b5d3/HalcoNET%205.8.0/Ventas/Program.cs), or [this](https://github.com/aaraujo2019/PEQMIMERIA/blob/e11b6d57e3bbbe505834cbfeb717fe87066d6850/DBMETAL_SHARP/DBMETAL_SHARP/Program.cs)) that won't be possible in this scenario, and will require deeper investigations and feasibility studies.

In the long term we could consider supporting a host/builder model, similar to the suggestion in [dotnet/winforms#2861](https://github.com/dotnet/winforms/issues/2861) and implementation in [alex-oswald/WindowsFormsLifetime](https://github.com/alex-oswald/WindowsFormsLifetime). E.g.:
```cs
[main: STAThread]
CreateHostBuilder().Build().Run();

public static IHostBuilder CreateHostBuilder() =>
    Host.CreateDefaultBuilder(Array.Empty<string>())
        .UseWindowsFormsLifetime<Form1>(options =>
        {
            options.HighDpiMode = HighDpiMode.SystemAware;
            options.EnableVisualStyles = true;
            options.CompatibleTextRenderingDefault = false;
            options.SuppressStatusMessages = false;
            options.EnableConsoleShutdown = true;
        })
        .ConfigureServices((hostContext, services) =>
        {
            // ...
        });
```
