# Combined sample browser

The app that hosts every demo is `MAUI/Combined/SampleBrowser.Maui.Combined`. Leave the upstream projects under `MAUI/<Control>/` unchanged. Changes belong in the host, `Directory.Build.props`, and `Directory.Build.targets`.

## Which build to run

Use a partial build when the sample libraries for that configuration and target framework are already on disk. That is the usual case, because sample sources are not part of the change.

Build the host and skip every sample project:

```bash
dotnet build MAUI/Combined/SampleBrowser.Maui.Combined/SampleBrowser.Maui.Combined.csproj -c Release -f net10.0-android -p:CombinedSampleBrowser=true -p:BuildProjectReferences=false -p:AndroidSdkDirectory="$HOME/Library/Android/sdk"
```

`-p:CombinedSampleBrowser=true` is required on a project build. The solution sets it from its name. A bare `dotnet build` of the host `.csproj` leaves it unset, and the document samples then fail `NU1902` and `NU1903` as errors.

`-p:BuildProjectReferences=false` does not recompile the samples. The host still copies their DLLs into the app. Those DLLs must already exist under each sample's `bin/<Configuration>/<TargetFramework>/`. Android sample libraries are written there without a runtime-identifier subdirectory. Expect a few minutes: packaging every library is the slow part.

One sample changed: build that project with `-p:CombinedSampleBrowser=true`, then build the host with `-p:BuildProjectReferences=false`.

`Directory.Build.props`, `Directory.Build.targets`, or missing sample output: build the solution so every sample compiles in library mode.

```bash
dotnet build SampleBrowser.Maui.Combined.sln -c Release -f net10.0-android -p:AndroidSdkDirectory="$HOME/Library/Android/sdk"
```

Replace `-c Release` and `-f net10.0-android` for Debug, `net10.0-ios`, or `net10.0-maccatalyst`. iOS and Mac Catalyst read the gitignored `signing.props` beside the host.

Windows is unpackaged. Pass the runtime identifier Meteor would pass:

```bash
dotnet build MAUI/Combined/SampleBrowser.Maui.Combined/SampleBrowser.Maui.Combined.csproj -c Debug -f net10.0-windows10.0.19041.0 -p:CombinedSampleBrowser=true -p:RuntimeIdentifier=win-x64 -p:WindowsPackageType=None
```

The executable is `bin/Debug/net10.0-windows10.0.19041.0/win-x64/SampleBrowser.Maui.Combined.exe`.

On Windows, iOS, and Mac Catalyst, `SkipFreshCombinedSamples` does not build a sample whose library for that configuration, framework, and runtime identifier is already newer than its sources and newer than `Directory.Build.props` and `Directory.Build.targets`. Android is excluded: the per-architecture pass must enter each sample to copy the already compiled library. `-p:CombinedBuildAllSamples=true` builds every sample. A change to either `Directory.Build` file makes every sample older than those files, so the next build compiles them all.

## Editor launch

`.vscode/launch.json` and `.vscode/tasks.json` are tracked. Launch **Combined: .NET Meteor Debugger**. Meteor fills configuration, target framework, runtime identifier, and device from the status bar, for Windows, Android, iOS, or Mac Catalyst. The task only adds `-p:CombinedSampleBrowser=true` and an empty `-p:WinUISDKReferences=`.

The empty WinUI property drops Meteor's Windows-only `-p:WinUISDKReferences=false`. The host compiles WinUI XAML and needs the SDK default. On Android and Apple that argument is absent, so the empty value is a no-op.

The same `tasks.json` has three Android Release tasks for the command palette: a full build, a host-only incremental build, and a deploy that installs the signed APK on every device `adb` lists. `AndroidSdkDirectory` is the Android Studio default for the OS running the task.

`.vscode/settings.json` points DotRush at `SampleBrowser.Maui.Combined.sln` with a path relative to the workspace. This repo has many solutions, so an empty list makes DotRush prompt and then write an absolute path.

`UseMonoRuntime=false` in `Directory.Build.props` applies only when `CombinedSampleBrowser` is set, `UseMonoRuntime` was not already set, and the runtime identifier starts with `win-`. Meteor's `dotnet build -p:RuntimeIdentifier=win-x64` makes restore evaluate every target framework with that identifier. Android and Apple then request `Microsoft.NETCore.App.Runtime.Mono.win-x64` and fail `NU1102`. Do not set `UseMonoRuntime` false for every platform.

## Install

Build once. Install the signed package on each device:

```bash
adb -s <serial> install -r MAUI/Combined/SampleBrowser.Maui.Combined/bin/Release/net10.0-android/net.prestonmccormick.syncfusion.maui.combined-Signed.apk
```

`adb devices -l` lists phones and running emulators. `emulator -list-avds` lists virtual devices. Start one with `emulator -avd <name>`.
