# AssetRipper Automation Plan

**Created**: 2025-11-14
**Status**: Ready for Implementation Phase
**Complexity**: Medium (4-8 hours estimated total)

---

## Executive Summary

AssetRipper is fundamentally **well-suited for automation** despite being a web-based GUI application. The core extraction libraries are completely separate from the GUI layer, and proof-of-concept non-interactive execution already exists in the SystemTester tool.

**Key Finding**: No major architectural changes are required. The solution is to create a dedicated CLI application that reuses the existing core libraries.

---

## 1. Repository Structure

### Directory Organization

```
/home/user/AssetRipper/
├── Source/                                    # All source code
│   ├── AssetRipper.Assets/                   # Core asset handling
│   ├── AssetRipper.Export/                   # Export core library
│   ├── AssetRipper.Export.PrimaryContent/    # Primary content export
│   ├── AssetRipper.Export.UnityProjects/     # Unity project export
│   ├── AssetRipper.Export.Modules.*/         # Export modules (Audio, Models, Textures)
│   ├── AssetRipper.GUI.Free/                 # Main GUI (web-based) - ENTRY POINT
│   ├── AssetRipper.GUI.Web/                  # Web framework/core
│   ├── AssetRipper.Import/                   # Import core library
│   ├── AssetRipper.Processing/               # Asset processing pipeline
│   ├── AssetRipper.Tools.SystemTester/       # ⭐ Non-interactive reference implementation
│   ├── AssetRipper.Tools.FileExtractor/
│   ├── AssetRipper.Tools.JsonSerializer/
│   └── [40+ additional projects for utilities, tests, generators]
├── AssetRipper.sln                           # Main solution file
└── docs/                                      # Documentation
```

### Key Technical Details
- **Framework**: .NET 8.0+
- **Project Type**: Mix of Console Apps, Libraries, and Web Application
- **Build Output**: `Source/0Bins/`
- **Configuration Format**: JSON (`AssetRipper.Settings.json`)

---

## 2. Entry Points & Call Paths

### Primary Entry Point: GUI.Web (Interactive)

**File**: `Source/AssetRipper.GUI.Free/Program.cs:3`

```
Program.cs:3
    ↓
WebApplicationLauncher.Launch(args)  [Line 39-68]
    ├─ Parses command-line arguments
    ├─ Loads configuration
    └─ WebApplicationLauncher.Launch(port, launchBrowser)  [Line 70-323]
        ├─ Initializes ASP.NET Core web server
        ├─ Registers HTTP endpoints (LoadFile, Export/UnityProject, Export/PrimaryContent)
        └─ app.Run()  [Line 323] ⚠️ BLOCKS HERE - runs indefinitely until shutdown
```

### Asset Extraction Call Chain (GUI.Web → Core)

1. **User loads file via web UI** → HTTP POST `/LoadFile`
2. **Commands.cs:36-59** → `HandleCommand<LoadFile>`
3. **GameFileLoader.LoadAndProcess()** [Line 54]
4. **ExportHandler.LoadAndProcess()** [Line 141]
   - Loads game files from paths
   - Runs asset processors (15+ processors for assembly, scene, texture, animation, prefab, etc.)
   - Returns `GameData` object
5. **ExportHandler.Export()** [Line 106]
   - Creates `ProjectExporter`
   - Exports complete Unity project structure

### Alternative Entry Point: SystemTester (Non-Interactive) ⭐

**File**: `Source/AssetRipper.Tools.SystemTester/Program.cs:13`

```
Main(args)
    ↓
Rip(args, outputPath)  [Line 27, 105-113]
    ├─ Create FullConfiguration(settings)
    ├─ ExportHandler exportHandler = new(settings)
    ├─ LoadAndProcess(inputPaths, fileSystem)
    ├─ PrepareExportDirectory(outputPath)  ← Deletes without confirmation
    └─ exportHandler.Export(gameData, outputPath, fileSystem)
        └─ Returns successfully, exits cleanly ✓
```

**This is the proven pattern for automated execution.**

---

## 3. Core Components

### Asset Extraction Pipeline

#### Loading Phase
- **Location**: `AssetRipper.Import`
- **Key Class**: `GameStructure`
- **Supported Formats**: Unity files (.assets, .bundle, .unity3d, CAB-* files)
- **Responsibility**: Parsing binary Unity asset structures

#### Processing Phase
- **Location**: `AssetRipper.Processing`
- **Processor Count**: ~15 different processors
- **Examples**:
  - Assembly processors (decompilation, repair)
  - Scene processors
  - Texture processors (sprite extraction, etc.)
  - Animation processors
  - Prefab processors
- **Responsibility**: Transform raw assets into exportable formats

#### Export Engines
Two distinct export modes:

1. **Unity Project Export** (Complete playable project)
   - **Class**: `ExportHandler` in `AssetRipper.Export.UnityProjects`
   - **Method**: `Export(gameData, path, fileSystem)`
   - **Output**: Full directory structure with assets, scripts, scene files
   - **Use Case**: Reverse engineering, asset reuse in new projects

2. **Primary Content Export** (Raw assets only)
   - **Class**: `PrimaryContentExporter` in `AssetRipper.Export.PrimaryContent`
   - **Method**: `Export(gameBundle, settings, fileSystem)`
   - **Output**: Textures (PNG, TGA), Audio (WAV, OGG), Models (FBX, OBJ)
   - **Use Case**: Asset archival, format conversion

### Configuration System

- **Location**: `AssetRipper.Export/Configuration/`
- **Main Class**: `FullConfiguration`
- **Storage**: JSON format in `AssetRipper.Settings.json`
- **Behavior**: Works with sensible defaults if no config file exists
- **Customizable**: Export settings, naming conventions, feature toggles

### Logging System

- **Location**: `AssetRipper.Import/Logging/`
- **Outputs**:
  - Console output via `ConsoleLogger`
  - File output via `FileLogger` (timestamped files)
- **Categories**: Import, Export, Processing, General, etc.
- **Behavior**: Already suitable for non-interactive logging

### GUI Layer (Web-Based Architecture)

⚠️ **Important**: Not traditional GUI (no WinForms/WPF)

- **Technology**: ASP.NET Core web application
- **Components**:
  - Web server hosting on localhost
  - Browser-based user interface
  - Optional native file/folder dialogs from `AssetRipper.NativeDialogs` NuGet package
- **Architecture Decision**: Core logic completely separated from web UI
  - Web layer: Request handlers, UI endpoints
  - Core layer: Extraction, processing, export (library form)

---

## 4. Automation Blockers

### CRITICAL Severity

#### 🔴 User Confirmation Dialog for Directory Deletion

**Location**: `Source/AssetRipper.GUI.Web/GameFileLoader.cs:63, 82`

```csharp
if (!await UserConsentsToDeletion())
{
    Logger.Info(LogCategory.Export, "User declined to delete existing export directory. Aborting export.");
    return;
}
```

- **Impact**: Blocks export if output directory exists and is non-empty
- **Current Behavior**: Waits indefinitely for user response
- **Dependency**: `ConfirmationDialog.Confirm()` from `AssetRipper.NativeDialogs`
- **Automation Workaround**: Skip confirmation check, always delete directory

#### 🔴 Web Server Blocking Execution

**Location**: `Source/AssetRipper.GUI.Web/WebApplicationLauncher.cs:323`

```csharp
app.Run();  // Blocks until server shutdown or Ctrl+C
```

- **Impact**: Application never exits naturally after export completion
- **Current Behavior**: Designed for interactive browser sessions
- **Automation Workaround**: Create separate CLI without web server

### HIGH Severity

#### 🟠 File/Folder Selection Dialogs

**Location**: `Source/AssetRipper.GUI.Web/Pages/Commands.cs:45-48, 73-76`

```csharp
else if (NativeDialog.Supported)
{
    paths = await OpenFileDialog.OpenFiles();  // GUI dialog
}
```

- **Impact**: Opens file picker if no path provided in HTTP request
- **Current Behavior**: Waits for user file selection
- **Automation Workaround**: Always provide paths via command-line arguments
- **Note**: Not an issue if CLI project provides paths directly

#### 🟠 Browser Auto-Launch

**Location**: `Source/AssetRipper.GUI.Web/WebApplicationLauncher.cs:125-135`

```csharp
if (launchBrowser)
{
    app.Lifetime.ApplicationStarted.Register(() => { OpenUrl(address); });
}
```

- **Impact**: Attempts to launch system browser on headless systems
- **Current Behavior**: May fail or hang on server environments
- **Automation Workaround**: Use `--launch-browser=false` (already implemented)
- **Status**: ✓ Already has CLI flag to disable

### LOW Severity

#### 🟡 Console.ReadKey() Calls

**Location**: Various tool programs (SystemTester, FileExtractor, etc.)

```csharp
Console.ReadKey();  // Prevents console auto-close
```

- **Impact**: Pauses execution waiting for any key press
- **Current Behavior**: Only affects interactive terminal sessions
- **Automation Workaround**: Detect environment (e.g., check if stdin is interactive)
- **Note**: SystemTester already has this pattern; can be conditional

---

## 5. Required Modifications

### RECOMMENDED APPROACH: Create New CLI Project

Create a dedicated `AssetRipper.CLI` project rather than modifying existing GUI code. This maintains separation of concerns and allows both GUI and CLI to coexist.

### Project Structure

```
Source/
├── AssetRipper.CLI/
│   ├── AssetRipper.CLI.csproj
│   ├── Program.cs
│   ├── CommandLineArguments.cs
│   ├── CommandLineParser.cs
│   └── readme.md
```

### Files to Create/Modify

#### 1. New Project File: `AssetRipper.CLI.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <PublishReadyToRun>true</PublishReadyToRun>
    <PublishSingleFile>true</PublishSingleFile>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="../AssetRipper.Import/AssetRipper.Import.csproj" />
    <ProjectReference Include="../AssetRipper.Processing/AssetRipper.Processing.csproj" />
    <ProjectReference Include="../AssetRipper.Export.UnityProjects/AssetRipper.Export.UnityProjects.csproj" />
    <ProjectReference Include="../AssetRipper.Export.PrimaryContent/AssetRipper.Export.PrimaryContent.csproj" />
    <ProjectReference Include="../AssetRipper.Export/AssetRipper.Export.csproj" />
  </ItemGroup>
</Project>
```

#### 2. New File: `CommandLineArguments.cs`

```csharp
namespace AssetRipper.CLI;

class CommandLineArguments
{
    public List<string> InputPaths { get; set; } = new();
    public string OutputPath { get; set; } = "";
    public ExportMode ExportMode { get; set; } = ExportMode.UnityProject;
    public string? ConfigFile { get; set; }
    public bool Overwrite { get; set; } = false;
    public string? LogFile { get; set; }
    public bool Verbose { get; set; } = false;
}

enum ExportMode
{
    UnityProject,
    PrimaryContent
}
```

#### 3. New File: `CommandLineParser.cs`

```csharp
namespace AssetRipper.CLI;

class CommandLineParser
{
    public static CommandLineArguments Parse(string[] args)
    {
        var options = new CommandLineArguments();

        for (int i = 0; i < args.Length; i++)
        {
            var arg = args[i];

            if (arg == "--input" || arg == "-i")
            {
                if (i + 1 < args.Length)
                    options.InputPaths.Add(args[++i]);
            }
            else if (arg == "--output" || arg == "-o")
            {
                if (i + 1 < args.Length)
                    options.OutputPath = args[++i];
            }
            else if (arg == "--export-mode")
            {
                if (i + 1 < args.Length)
                    options.ExportMode = Enum.Parse<ExportMode>(args[++i], ignoreCase: true);
            }
            else if (arg == "--config")
            {
                if (i + 1 < args.Length)
                    options.ConfigFile = args[++i];
            }
            else if (arg == "--overwrite")
            {
                options.Overwrite = true;
            }
            else if (arg == "--log")
            {
                if (i + 1 < args.Length)
                    options.LogFile = args[++i];
            }
            else if (arg == "--verbose" || arg == "-v")
            {
                options.Verbose = true;
            }
        }

        return options;
    }
}
```

#### 4. New File: `Program.cs`

```csharp
using AssetRipper.Export.Configuration;
using AssetRipper.Export.PrimaryContent;
using AssetRipper.Export.UnityProjects;
using AssetRipper.Import.Logging;
using AssetRipper.IO.Files;

namespace AssetRipper.CLI;

class Program
{
    static int Main(string[] args)
    {
        try
        {
            var options = CommandLineParser.Parse(args);

            if (!ValidateArguments(options))
            {
                PrintUsage();
                return 1;
            }

            // Setup logging
            SetupLogging(options);

            Logger.LogSystemInformation("AssetRipper CLI");
            Logger.Info(LogCategory.General, $"Input paths: {string.Join(", ", options.InputPaths)}");
            Logger.Info(LogCategory.General, $"Output path: {options.OutputPath}");
            Logger.Info(LogCategory.General, $"Export mode: {options.ExportMode}");

            // Load configuration
            FullConfiguration settings = LoadConfiguration(options.ConfigFile);

            // Prepare export directory
            PrepareExportDirectory(options.OutputPath, options.Overwrite);

            // Execute extraction
            Logger.Info(LogCategory.General, "Starting asset extraction...");

            ExportHandler handler = new(settings);
            GameData gameData = handler.LoadAndProcess(options.InputPaths, LocalFileSystem.Instance);

            Logger.Info(LogCategory.General, "Asset processing completed");
            Logger.Info(LogCategory.Export, "Starting export...");

            // Export based on mode
            if (options.ExportMode == ExportMode.UnityProject)
            {
                handler.Export(gameData, options.OutputPath, LocalFileSystem.Instance);
            }
            else
            {
                var exporter = PrimaryContentExporter.CreateDefault(gameData, settings);
                exporter.Export(gameData.GameBundle, settings, LocalFileSystem.Instance);
            }

            Logger.Info(LogCategory.General, "✓ Export completed successfully");
            return 0;
        }
        catch (Exception ex)
        {
            Logger.Error(LogCategory.General, $"✗ Export failed: {ex.Message}", ex);
            return 1;
        }
    }

    static bool ValidateArguments(CommandLineArguments options)
    {
        if (options.InputPaths.Count == 0)
        {
            Logger.Error(LogCategory.General, "No input paths provided");
            return false;
        }

        if (string.IsNullOrEmpty(options.OutputPath))
        {
            Logger.Error(LogCategory.General, "No output path specified");
            return false;
        }

        foreach (var inputPath in options.InputPaths)
        {
            if (!File.Exists(inputPath) && !Directory.Exists(inputPath))
            {
                Logger.Error(LogCategory.General, $"Input path does not exist: {inputPath}");
                return false;
            }
        }

        return true;
    }

    static void SetupLogging(CommandLineArguments options)
    {
        Logger.Add(new ConsoleLogger(verbose: options.Verbose));

        if (!string.IsNullOrEmpty(options.LogFile))
        {
            var logDir = Path.GetDirectoryName(options.LogFile);
            if (!string.IsNullOrEmpty(logDir) && !Directory.Exists(logDir))
                Directory.CreateDirectory(logDir);

            Logger.Add(new FileLogger(options.LogFile));
        }
    }

    static FullConfiguration LoadConfiguration(string? configPath)
    {
        if (!string.IsNullOrEmpty(configPath) && File.Exists(configPath))
        {
            Logger.Info(LogCategory.General, $"Loading configuration from: {configPath}");
            return FullConfiguration.LoadFromFile(configPath);
        }

        Logger.Info(LogCategory.General, "Using default configuration");
        return new FullConfiguration();
    }

    static void PrepareExportDirectory(string path, bool overwrite)
    {
        if (Directory.Exists(path))
        {
            if (overwrite)
            {
                Logger.Info(LogCategory.Export, $"Deleting existing directory: {path}");
                try
                {
                    Directory.Delete(path, recursive: true);
                }
                catch (Exception ex)
                {
                    throw new InvalidOperationException($"Failed to delete existing directory: {path}", ex);
                }
            }
            else
            {
                throw new InvalidOperationException(
                    $"Output directory already exists: {path}\n" +
                    $"Use --overwrite flag to replace existing directory.");
            }
        }

        try
        {
            Directory.CreateDirectory(path);
            Logger.Info(LogCategory.Export, $"Created export directory: {path}");
        }
        catch (Exception ex)
        {
            throw new InvalidOperationException($"Failed to create output directory: {path}", ex);
        }
    }

    static void PrintUsage()
    {
        Console.WriteLine("""
            AssetRipper CLI - Non-interactive asset extraction tool

            USAGE:
                AssetRipper.CLI [OPTIONS]

            REQUIRED OPTIONS:
                --input <path> (-i)        Input file or directory (can be repeated)
                --output <path> (-o)       Output directory for extracted assets

            OPTIONAL OPTIONS:
                --export-mode <mode>       Export mode: UnityProject (default) or PrimaryContent
                --config <path>            Path to configuration JSON file
                --overwrite                Delete existing output directory if it exists
                --log <path>               Write detailed log to file
                --verbose (-v)             Enable verbose console output

            EXAMPLES:
                # Extract to Unity project
                AssetRipper.CLI --input game.exe --output ./extracted --overwrite

                # Extract primary content with logging
                AssetRipper.CLI -i game.apk -o ./assets --export-mode PrimaryContent --log ./export.log

                # Batch extraction with custom config
                AssetRipper.CLI -i ./games -o ./extracted --config settings.json --verbose
            """);
    }
}
```

#### 5. Update Solution File: `AssetRipper.sln`

Add reference to new project in the solution file.

---

## 6. Implementation Sequence

### Phase 1: CLI Core Implementation (Est. 2-3 hours)

**Task 1.1**: Create CLI project structure [1 hour]
- [ ] Create `AssetRipper.CLI/` directory
- [ ] Create `AssetRipper.CLI.csproj` with correct dependencies
- [ ] Add to `AssetRipper.sln`
- [ ] Verify project builds

**Task 1.2**: Implement argument parsing [30 min]
- [ ] Create `CommandLineArguments.cs` class
- [ ] Create `CommandLineParser.cs` with full argument support
- [ ] Test with various argument combinations

**Task 1.3**: Implement core execution logic [1 hour]
- [ ] Create `Program.cs` with extraction flow
- [ ] Copy proven pattern from `SystemTester`
- [ ] Integrate logging
- [ ] Handle directory deletion without user confirmation
- [ ] Implement proper exit codes (0 = success, 1 = error)

**Task 1.4**: Error handling and validation [30 min]
- [ ] Add comprehensive try-catch blocks
- [ ] Implement input validation
- [ ] Add meaningful error messages
- [ ] Handle edge cases (missing files, permissions, etc.)

### Phase 2: Testing & Refinement (Est. 1-2 hours)

**Task 2.1**: Unit testing [1 hour]
- [ ] Test argument parsing with various inputs
- [ ] Test with real Unity game files
- [ ] Test both export modes (UnityProject and PrimaryContent)
- [ ] Test error conditions (missing input, invalid output path)
- [ ] Test directory overwrite behavior

**Task 2.2**: Integration testing [30 min]
- [ ] End-to-end extraction flow
- [ ] Logging output verification
- [ ] Performance baseline measurement

**Task 2.3**: Documentation [30 min]
- [ ] Create README.md for CLI project
- [ ] Document all command-line options
- [ ] Provide usage examples

### Phase 3: Automation Wrapper Scripts (Est. 1-2 hours)

**Task 3.1**: Linux/Unix automation [45 min]
- [ ] Create bash script wrapper: `/opt/assetripper/extract.sh`
- [ ] Setup cron job for scheduled execution
- [ ] Implement log rotation
- [ ] Add error notification mechanism

**Task 3.2**: Windows automation [45 min]
- [ ] Create PowerShell wrapper script
- [ ] Setup Task Scheduler entry
- [ ] Implement log management
- [ ] Add error alerts

**Task 3.3**: Docker containerization (Optional) [30 min]
- [ ] Create Dockerfile
- [ ] Setup volume mounts for input/output
- [ ] Document container usage

### Phase 4: Integration & Release (Est. 30 min)

**Task 4.1**: Solution integration [20 min]
- [ ] Add CLI project to release build pipeline
- [ ] Verify all dependencies compile
- [ ] Test release build

**Task 4.2**: Documentation update [10 min]
- [ ] Update main README.md
- [ ] Add CLI section to documentation
- [ ] Include automation examples

---

## 7. Technical Deep Dives Required

### Assembly Loading & Runtime

**Question**: How do assembly processors behave in non-GUI environment?

- **Files**: `AssetRipper.Processing/` (Assembly-related processors)
- **Risk**: Managed assembly loading might fail without proper AppDomain setup
- **Action**: Test with real game files containing managed assemblies
- **Mitigation**: May need to add error recovery or skip problematic assemblies

### External Resource Dependencies

**Question**: What are the `EngineResourceData` requirements?

- **Location**: `AssetRipper.Export.Configuration/EngineResourceData.cs`
- **Purpose**: Engine-specific data for asset type handling
- **Risk**: May require embedded resources or external files
- **Action**: Verify these are packaged with executable
- **Testing**: Confirm CLI can access them in different installation locations

### Premium Edition Features

**Question**: Are there premium-only features blocking automation?

- **Location**: `AssetRipper.GUI.Web/GameFileLoader.cs:39` (Premium check)
- **Impact**: Some features may differ between Free and Premium
- **Action**: Test with both edition types if applicable
- **Note**: CLI can be built for either edition

### Performance Characteristics

**Question**: Memory and CPU requirements for large games?

- **Testing needed**: Extract with games of various sizes (1GB-100GB+)
- **Metrics**: Memory usage, CPU utilization, processing time
- **Optimization**: Identify bottlenecks for scheduled environment
- **Planning**: Resource allocation for automation scheduling

---

## 8. Next Actions (For Primary Agent)

### Immediate (High Priority)

1. **Create CLI Project Structure**
   - [ ] Create `Source/AssetRipper.CLI/` directory
   - [ ] Implement `Program.cs` following SystemTester pattern
   - [ ] Implement `CommandLineArguments.cs` and `CommandLineParser.cs`
   - [ ] Add to solution file

2. **Initial Build & Test**
   - [ ] Verify project builds cleanly
   - [ ] Test with a sample Unity game
   - [ ] Validate both export modes work

3. **Create Test Harness**
   - [ ] Prepare test game files
   - [ ] Document expected outputs
   - [ ] Create test scripts for validation

### Secondary (After Initial Implementation)

4. **Automation Wrapper Scripts**
   - [ ] Create Linux cron/systemd examples
   - [ ] Create Windows Task Scheduler examples
   - [ ] Document scheduling setup

5. **Enhanced Documentation**
   - [ ] CLI usage guide
   - [ ] Automation setup guide
   - [ ] Troubleshooting section

6. **Performance Optimization** (If needed)
   - [ ] Profile extraction with large games
   - [ ] Identify and optimize bottlenecks
   - [ ] Document resource requirements

### Deferred (Future Phases)

7. **Advanced Features** (Optional)
   - [ ] Batch processing support
   - [ ] Resume capability for large extractions
   - [ ] Incremental extraction (only changed files)
   - [ ] Progress reporting via HTTP API

8. **CI/CD Integration**
   - [ ] Add CLI to release pipeline
   - [ ] Create standalone executable artifacts
   - [ ] Add automated testing for CLI

---

## 9. Code Templates & Examples

### Usage Examples

#### Basic Unity Project Export
```bash
./AssetRipper.CLI \
    --input game.exe \
    --output ./extracted \
    --overwrite
```

#### Primary Content Export with Logging
```bash
./AssetRipper.CLI \
    --input game.apk \
    --output ./assets \
    --export-mode PrimaryContent \
    --log ./extraction.log \
    --verbose
```

#### Batch Processing Multiple Games
```bash
for game in /data/games/*.exe; do
    output_dir="/data/extracted/$(basename "$game" .exe)"
    ./AssetRipper.CLI \
        --input "$game" \
        --output "$output_dir" \
        --overwrite \
        --log "./logs/$(date +%Y%m%d_%H%M%S).log"
done
```

### Cron Schedule Example

```cron
# Run extraction every night at 2 AM
0 2 * * * /opt/assetripper/batch_extract.sh >> /var/log/assetripper/cron.log 2>&1
```

### PowerShell Scheduled Task Example

```powershell
$action = New-ScheduledTaskAction `
    -Execute "C:\AssetRipper\AssetRipper.CLI.exe" `
    -Argument '--input D:\games --output D:\extracted --overwrite'

$trigger = New-ScheduledTaskTrigger -Daily -At 2am

Register-ScheduledTask `
    -TaskName "AssetRipperNightly" `
    -Action $action `
    -Trigger $trigger `
    -User "SYSTEM"
```

---

## 10. Risk Assessment & Mitigation

| Risk | Severity | Mitigation |
|------|----------|-----------|
| Assembly loading fails without GUI | HIGH | Test thoroughly; implement error recovery |
| Resource data unavailable at runtime | MEDIUM | Package resources with executable |
| Large game extraction hangs | MEDIUM | Add progress reporting; implement timeouts |
| Output directory permissions error | LOW | Validate paths before execution; clear error messages |
| Encoding issues in asset names | LOW | Use UTF-8 throughout; handle invalid chars |

---

## 11. Success Criteria

Implementation is complete when:

- [ ] CLI application builds without errors
- [ ] Successfully extracts assets from sample Unity games
- [ ] Both export modes (UnityProject, PrimaryContent) produce correct output
- [ ] Proper exit codes returned (0 = success, non-zero = failure)
- [ ] Can run without any user interaction or dialogs
- [ ] Handles errors gracefully with meaningful messages
- [ ] Works in headless/server environment
- [ ] Logging output is complete and useful
- [ ] Can be scheduled via cron/Task Scheduler
- [ ] Documentation complete and tested

---

## Summary

**AssetRipper is suitable for automation.** The core extraction libraries are well-designed for non-interactive use. The recommended approach is to create a dedicated CLI project that:

1. Reuses existing core libraries (`ExportHandler`, processing pipeline)
2. Provides command-line interface for automation
3. Handles all user confirmations automatically
4. Returns proper exit codes for orchestration
5. Produces detailed logs for monitoring

**Estimated Implementation Time**: 4-8 hours total
**Complexity**: Medium
**Risk Level**: Low (proven pattern exists in SystemTester)

No major architectural changes are required. All blockers are in the GUI layer, not the core extraction engine.

---

**Next Step**: Begin Phase 1 implementation by creating the CLI project structure and initial Program.cs
