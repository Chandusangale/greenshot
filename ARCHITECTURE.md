# Greenshot Architecture Documentation

## Overview

Greenshot is a Windows-based screenshot tool built using a **plugin-based architecture** with a modular design. This document describes the architectural patterns, components, and technologies used in the project.

## Technology Stack

### Core Technologies
- **Framework**: .NET Framework 4.7.2
- **UI Framework**: Windows Forms (WinForms) with WPF support
- **Programming Language**: C# (latest language version)
- **Build System**: MSBuild / .NET SDK
- **Logging**: log4net 2.0.15

### Key Libraries
- **Dapplo.Windows.***: Suite of Windows API wrappers for Clipboard, DPI, GDI32, Icons, Kernel32, and Multimedia
- **Dapplo.HttpExtensions.JsonNet**: HTTP and JSON handling
- **HtmlAgilityPack**: HTML parsing
- **Svg**: SVG rendering support

## Architectural Patterns

### 1. Plugin Architecture

Greenshot uses a **plugin-based architecture** that allows extensibility through dynamically loaded plugins.

#### Plugin System Components

**Core Interface**: `IGreenshotPlugin`
```csharp
public interface IGreenshotPlugin : IDisposable
{
    bool Initialize();
    void Shutdown();
    void Configure();
    string Name { get; }
    bool IsConfigurable { get; }
}
```

**Plugin Host**: `IGreenshotHost`
- Manages plugin lifecycle
- Provides services to plugins
- Located in `Greenshot.Helpers.PluginHelper`

#### Available Plugins

The project includes multiple built-in plugins, each in its own assembly:

1. **Greenshot.Plugin.ExternalCommand** - Execute external commands with screenshots
2. **Greenshot.Plugin.Box** - Upload to Box cloud storage
3. **Greenshot.Plugin.Imgur** - Upload to Imgur
4. **Greenshot.Plugin.Dropbox** - Upload to Dropbox
5. **Greenshot.Plugin.Flickr** - Upload to Flickr
6. **Greenshot.Plugin.Jira** - Integration with Jira
7. **Greenshot.Plugin.Office** - Integration with Microsoft Office
8. **Greenshot.Plugin.Win10** - Windows 10 specific features
9. **Greenshot.Plugin.Confluence** - Integration with Confluence
10. **Greenshot.Plugin.GooglePhotos** - Upload to Google Photos
11. **Greenshot.Plugin.Photobucket** - Upload to Photobucket

### 2. Destination Pattern

The **Destination Pattern** is used for handling screenshot outputs through the `IDestination` interface.

**Key Features**:
- Each destination can handle captured screenshots
- Destinations can be static or dynamic
- Support for priority-based ordering
- Menu integration support
- Export information tracking

```csharp
// Simplified interface - see IDestination.cs for complete definition
public interface IDestination : IDisposable, IComparable
{
    string Designation { get; }
    string Description { get; }
    int Priority { get; }
    Image DisplayIcon { get; }
    bool IsActive { get; }
    bool IsDynamic { get; }
    bool UseDynamicsOnly { get; }
    bool IsLinkable { get; }
    Keys EditorShortcutKeys { get; }
    
    IEnumerable<IDestination> DynamicDestinations();
    ToolStripMenuItem GetMenuItem(bool addDynamics, ContextMenuStrip menu, EventHandler destinationClickHandler);
    ExportInformation ExportCapture(bool manuallyInitiated, ISurface surface, ICaptureDetails captureDetails);
}
```

**Note**: `GetMenuItem` creates and returns a menu item that represents this destination in the UI.

### 3. Service Locator Pattern

Greenshot uses a simple **Service Locator** pattern for dependency management.

**Implementation**: `SimpleServiceProvider`
- Lightweight dependency injection container
- Singleton instance accessible via `SimpleServiceProvider.Current`
- Type-based service registration and resolution
- Supports multiple instances per type

```csharp
public interface IServiceLocator
{
    TService GetInstance<TService>();
    IReadOnlyList<TService> GetAllInstances<TService>();
    void AddService<TService>(IEnumerable<TService> services);
    void AddService<TService>(params TService[] services);
}
```

### 4. Processor Pattern

The **Processor Pattern** handles post-capture processing through `IProcessor` interface:
- Processes screenshots after capture
- Chain of responsibility for multiple processors
- Example: `TitleFixProcessor` for modifying window titles

### 5. Capture Pattern

Screenshots are handled through specialized interfaces:
- **ICapture**: Represents a captured screenshot
- **ICaptureDetails**: Metadata about the capture
- **ISurface**: Drawing surface for editing screenshots
- **ICaptureHelper**: Utilities for capturing screenshots

## Project Structure

### Core Components

```
src/
├── Greenshot/                    # Main application
│   ├── Forms/                    # UI forms (MainForm, SettingsForm, etc.)
│   ├── Helpers/                  # Helper classes (PluginHelper, CaptureHelper, etc.)
│   ├── Destinations/             # Built-in destinations
│   ├── Processors/               # Screenshot processors
│   └── Configuration/            # Configuration management
│
├── Greenshot.Base/               # Core library
│   ├── Interfaces/               # Core interfaces
│   │   ├── Plugin/              # Plugin-related interfaces
│   │   ├── Drawing/             # Drawing interfaces
│   │   └── Forms/               # Form interfaces
│   ├── Core/                     # Core implementations
│   │   ├── SimpleServiceProvider.cs
│   │   ├── AbstractDestination.cs
│   │   ├── AbstractProcessor.cs
│   │   └── CaptureHandler.cs
│   └── Controls/                 # Reusable controls
│
├── Greenshot.Editor/             # Image editor component
│   ├── Drawing/                  # Drawing elements
│   ├── Forms/                    # Editor forms
│   ├── Memento/                  # Undo/Redo pattern
│   └── Helpers/                  # Editor utilities
│
└── Greenshot.Plugin.*/           # Individual plugins
```

### Layer Architecture

```
┌─────────────────────────────────────┐
│     Greenshot (Main Application)    │
│  - Windows Forms UI                 │
│  - Application Entry Point          │
│  - Main Workflow Coordination       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│      Greenshot.Editor               │
│  - Image Editing Capabilities       │
│  - Drawing Tools                    │
│  - Memento Pattern (Undo/Redo)      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│      Greenshot.Base                 │
│  - Core Interfaces                  │
│  - Service Locator                  │
│  - Common Utilities                 │
│  - Plugin Infrastructure            │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│         Plugins (Optional)          │
│  - Upload Destinations              │
│  - External Integrations            │
│  - Extended Functionality           │
└─────────────────────────────────────┘
```

## Key Design Decisions

### 1. Modular Plugin System
- **Why**: Allows extensibility without modifying core code
- **How**: Interface-based contracts, dynamic assembly loading
- **Benefits**: Easy to add new features, optional functionality

### 2. Service Locator over DI Container
- **Why**: Lightweight, simple, no external dependencies for DI
- **How**: Custom `SimpleServiceProvider` implementation
- **Trade-offs**: Less type safety than modern DI containers, but sufficient for project needs

### 3. Windows Forms for UI
- **Why**: Native Windows integration, mature framework
- **How**: Traditional event-driven Windows Forms with some WPF components
- **Benefits**: Direct access to Windows APIs, good performance

### 4. Interface Segregation
- **Why**: Follows SOLID principles
- **How**: Multiple small, focused interfaces (ICapture, ISurface, IDestination, etc.)
- **Benefits**: Flexible implementations, testable code

## Configuration Management

Configuration is handled through:
- **IniFile-based** configuration system
- **CoreConfiguration** class for application settings
- Plugin-specific configuration classes
- Settings forms for user configuration

## Event Flow

### Typical Screenshot Capture Flow

```
1. User triggers capture (hotkey/menu)
   ↓
2. CaptureHelper creates capture
   ↓
3. Capture is wrapped in ICapture interface
   ↓
4. Processors run on capture (optional)
   ↓
5. Editor opens (if configured) via ISurface
   ↓
6. User selects destination
   ↓
7. Destination.ExportCapture() is called
   ↓
8. Export result is provided as ExportInformation
```

## Extension Points

Developers can extend Greenshot through:

1. **Custom Plugins**: Implement `IGreenshotPlugin` to add new functionality
2. **Custom Destinations**: Implement `IDestination` to add new export targets
3. **Custom Processors**: Implement `IProcessor` to add post-capture processing
4. **Custom File Formats**: Implement `IFileFormatHandler` to support additional image formats
   - Three action types: SaveToStream, LoadFromStream, LoadDrawableFromStream
   - Each handler registers which file extensions it supports for each action type

## Build Configuration

- **Target Framework**: net472 (.NET Framework 4.7.2)
- **Platform**: Windows (win10-x64, win10-x86, win-x64, win-x86)
- **Minimum Windows Version**: Windows 7 or later (via .NET Framework 4.7.2 requirement)
- **Output**: Windows Executable (WinExe)
- **Installer**: InnoSetup-based installer
- **Versioning**: Using Nerdbank.GitVersioning (version.json)

## Threading Model

- **Main Thread**: UI operations, Windows Forms event handling
- **Background Threads**: Used for uploads, network operations
- **Synchronization**: UI marshalling for cross-thread operations

## Memory Management

- **IDisposable Pattern**: Used throughout for proper resource cleanup
- **Image Handling**: Custom FastBitmap for performance
- **Bitmap Management**: Careful tracking of GDI+ object lifetimes

## Security Considerations

- **TLS Support**: TLS 1.2 enabled for secure communications with cloud services
- **Credential Storage**: Credentials are securely handled via `CredentialsHelper` which uses Windows Credential Manager for secure storage
- **OAuth Support**: Built-in OAuth implementation for cloud services (Imgur, Dropbox, Flickr, etc.)

## Testing Strategy

The architecture supports testing through:
- Interface-based design
- Service locator for dependency injection in tests
- Separation of concerns between UI and logic

## Future Considerations

The plugin architecture allows for:
- Additional cloud storage integrations
- New export formats
- Enhanced image processing
- Integration with other tools and services

## References

- Main repository: https://github.com/greenshot/greenshot
- Official website: https://getgreenshot.org/
- License: GNU General Public License v3.0
