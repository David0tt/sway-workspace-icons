# sway Workspace Icons

![Example swaybar with workspace icons](images/bar_example_image.png)
*Example: swaybar showing workspaces with icons for currently running programs*

Dynamically update sway workspace names with program icons of currently running programs. Since swaybar cannot display images directly, this daemon creates a custom font on-the-fly from system icon files that swaybar can render. 

## Features
- **Original color icons**: Displays the correct icons for any program (as they appear in desktop environments like GNOME)
- **Fast updates**: Icons in workspace names update automatically when programs change or move between workspaces
- **SVG and PNG support**: Handles both SVG (preferred) and PNG icon formats
- **Program count indicators**: When multiple instances of the same program are running, their count can be shown with subscripts or superscripts


# Installation

## Prerequisites

Most systems already have these installed:
- `fontconfig`
- `libcairo2`

On Debian/Ubuntu:
```bash
sudo apt install fontconfig libcairo2
```

## Quick Install

### Install Python Package

```bash
git clone https://github.com/David0tt/sway-workspace-icons/
pip install sway-workspace-icons/
```

### sway Configuration

Add to your `~/.config/sway/config`:

```
# Change the swaybar font
bar {
    font swayWorkspaceDaemonIconFont 20
    height 30  # Required to prevent incorrect bar height
}

# Auto-start the daemon
exec_always --no-startup-id sway-workspace-icon-daemon

# Use workspace numbers in keybindings
# (Standard config uses workspace names, which won't work since names change dynamically)
bindsym $mod+1 workspace number 1
bindsym $mod+2 workspace number 2
# ... continue for other workspaces
```

## Using Conda for Environment Management (Recommended)

We create an isolated Python environment:

```bash
conda create -n swayWorkspaceIcons python==3.12 -y
conda activate swayWorkspaceIcons
git clone https://github.com/David0tt/sway-workspace-icons/
pip install sway-workspace-icons/
```

Then add to your `~/.config/sway/config`:

```
# Change the swaybar font
bar {
    font swayWorkspaceDaemonIconFont 20
    height 30
}

# Auto-start the daemon with conda environment
exec_always --no-startup-id bash -c "source ~/miniconda3/bin/activate swayWorkspaceIcons && exec sway-workspace-icon-daemon"

# Use workspace numbers in keybindings
bindsym $mod+1 workspace number 1
bindsym $mod+2 workspace number 2
# ... continue for other workspaces
```

In waybar, you need to instead change the font in the styling in `~/.config/waybar/style.css` e.g.:

```
#workspaces button {
    font-family: "swayWorkspaceDaemonIconFont", monospace;
}
```



## Usage

After installation, the daemon can be run manually:

```bash
sway-workspace-icon-daemon
```

For automatic startup, add the appropriate `exec_always` line to your sway config as shown in the installation section.

### Command-line Options

```bash
sway-workspace-icon-daemon --help                     # Show all options
sway-workspace-icon-daemon --verbose                  # Enable debug logging
sway-workspace-icon-daemon --full-rebuild             # Rebuild icon map from scratch (useful for fixing errors)
sway-workspace-icon-daemon --unique-icons MODE        # Display mode: numbers_superscript|numbers_subscript|unique|nonunique
sway-workspace-icon-daemon --no-placeholder-icon      # Don't use placeholder icons when program icons are not found
```

## Uninstall

### Remove the Package

```bash
pip uninstall sway-workspace-icons
```

### Restore sway Configuration

Change the font in your sway config from:

```
bar {
    font swayWorkspaceDaemonIconFont 20
    height 30
}
```

back to the default (or your preferred font):

```
bar {
    font pango:monospace 18
}
```

Remove the autostart lines

    exec_always --no-startup-id sway-workspace-icon-daemon

or

    exec_always --no-startup-id bash -c "source ~/miniconda3/bin/activate swayWorkspaceIcons && exec sway-workspace-icon-daemon"


### Clean Up User Data (Optional)

```bash
rm -rf ~/.config/swaybar-workspace-program-icons
rm -rf ~/.cache/swaybar-workspace-program-icons
rm -f ~/.local/share/fonts/swayWorkspaceDaemonIconFont.ttf
```




## Files and Directories

The daemon follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html):

### Configuration
- **`~/.config/swaybar-workspace-program-icons/program_icon_map.yaml`**
  - Auto-generated mapping of programs to icons and Unicode codepoints
  - Persists across daemon restarts

### Cache
- **`~/.cache/swaybar-workspace-program-icons/swayWorkspaceDaemonIconFont.ttf`**
  - Generated custom font containing program icons
  - Can be regenerated with the `--rebuild` or `--full-rebuild` flags
  - Also installed to `~/.local/share/fonts/` for system-wide access

### Environment Variables
The daemon respects `$XDG_CONFIG_HOME` and `$XDG_CACHE_HOME` if set. Override default locations with command-line arguments:
```bash
sway-workspace-icon-daemon --program-icon-map /custom/path/map.yaml
sway-workspace-icon-daemon --font-output /custom/path/font.ttf
```

## Inspiration

This project is inspired by:
- [i3-workspace-names-daemon](https://github.com/cboddy/i3-workspace-names-daemon)
- [i3scripts/autoname_workspaces.py](https://github.com/justbuchanan/i3scripts)

Unlike these projects, which rely on pre-existing icon fonts (like FontAwesome or Nerd Fonts) with predefined program-to-icon mappings, this daemon uses actual program icons from your system. Therefore in contrast to these other tools it can:
- show color icons
- show icons for almost any program automatically. If you encounter a program, where no icon or the wrong icon is shown, please let me know in an issue. 


## Known Limitations

- **swaybar restart required**: When installing a new font, swaybar must be stopped (to prevent crashes), restarted, and then restarted again after the font cache updates. This causes brief flickering or temporary disappearance of swaybar on slower machines. This only occurs when a new program is encountered for the first time.
- **Icon spacing with counts**: Numbers indicating program counts use subscripts/superscripts, which disrupt equal spacing between icons. A better solution might use Unicode diacritics or embed numbers directly in icons, but this has not been implemented yet. 


## How It Works

Since swaybar cannot display images directly, this daemon creates a custom font from program icons on-the-fly:

1. The daemon monitors sway window events (new, close, move)
2. When a new program is detected:
   - Finds the program's `.desktop` file
   - Extracts the `Icon=` entry to locate the icon file in standard system directories
   - Assigns the program a Unicode codepoint in the Private Use Area (PUA)
   - Rebuilds the custom font with all known icons
   - Stops swaybar (to prevent crashes when modifying the active font)
   - Installs the font to `~/.local/share/fonts/`
   - Restarts swaybar
   - Reloads the font cache with `fc-cache`
   - Restarts swaybar again to load the updated cache
3. Workspace names are updated with icon characters from the custom font whenever window events occur
4. A program-to-icon map is persisted for consistency and fast restarts. The font is only rebuilt when a new program is encountered


## Development

### Architecture

The project consists of three main components:

**`font_builder.py - FontBuilder`**
- Builds icon fonts from a base font (NotoColorEmoji) and icon files (PNG/SVG)
- Can be used standalone:
  ```bash
  python font_builder.py --remove-original-symbols --icon-paths /usr/share/icons/hicolor/256x256/apps/firefox.png
  ```

**`sway_workspace_icon_daemon.py - ProgramIconMap`**
- Manages the relationship between program names, icon paths, and Unicode codepoints
- Persists mappings to `program_icon_map.yaml`

**`sway_workspace_icon_daemon.py - WorkspaceIconDaemon`**
- Monitors sway window events (open, close, move) via i3ipc
- Discovers icon locations for programs
- Updates the ProgramIconMap
- Rebuilds and installs font when new programs are detected
- Updates workspace names dynamically

### Testing

Test the font builder:

```bash
cd sway_workspace_icons
python font_builder.py --remove-original-symbols \
    --icon-paths /usr/share/icons/hicolor/256x256/apps/firefox.png \
                 /usr/share/icons/hicolor/scalable/apps/gvim.svg
```

Examine the generated font:

```bash
font-manager MyCreatedIconFont.ttf
```

Verify that icons are created at the correct Unicode locations (Private Use Area starting at U+E000).

### Possible Future Features

- [ ] Limit maximum number of icons shown per workspace
- [ ] Better icon spacing with count indicators
- [ ] support for different bars. In theory, this works, with any bar that shows workspaces by their title, and where you can set the font (and that has a reasonable font rendering support, e.g. for emojis). However, the updating sequence needs to be modified, depending on the bar. 