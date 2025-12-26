# Claude.md - Project Analysis & Improvement Roadmap

## Project Overview

**egui-rad-builder** is a Rapid Application Development (RAD) GUI builder tool for the egui immediate-mode GUI framework. It allows developers to visually design user interfaces through drag-and-drop, then generates production-ready Rust code for egui-based applications.

**Current Version:** 0.1.10
**License:** MIT
**Status:** Active early development

---

## Recent Changes (2025-12-26)

### New Widgets Added (5 new types)
- **TextArea** - Multi-line text editing
- **DragValue** - Compact numeric input with drag-to-adjust
- **Spinner** - Loading/progress indicator
- **ColorPicker** - RGBA color selection with picker UI
- **Code** - Code editor with monospace font and syntax styling

### Keyboard Shortcuts Implemented
| Shortcut | Action |
|----------|--------|
| `Delete` / `Backspace` | Delete selected widget |
| `Ctrl+C` | Copy selected widget |
| `Ctrl+V` | Paste widget |
| `Ctrl+D` | Duplicate selected widget |
| `Ctrl+G` | Generate code |

### UX Improvements
- Widget clipboard for copy/paste operations
- Updated Tips panel with shortcuts reference

---

## Architecture Analysis

### Codebase Structure

```
src/
├── main.rs        (47 lines)   - Entry point, window initialization
├── app.rs         (1668 lines) - Core application logic (needs splitting)
├── project.rs     (26 lines)   - Project data model
└── widget/
    └── mod.rs     (119 lines)  - Widget types and utilities
```

**Total:** ~1,860 lines of Rust

### Design Pattern

The codebase follows an MVC-style architecture:
- **Model:** `Project` and `Widget` types define the data
- **View:** GUI rendering in `preview_panels_ui()`, `draw_widget()`, palette/inspector UI
- **Controller:** Event handling and state management in `RadBuilderApp`

### Supported Widgets (24 types)

**Basic:** Label, Button, ImageTextButton, Checkbox, Link, Hyperlink, SelectableLabel, Separator

**Input:** TextEdit, TextArea, Password, Slider, DragValue, ComboBox, RadioGroup, DatePicker, AngleSelector, ColorPicker

**Advanced:** MenuButton, CollapsingHeader, Tree, ProgressBar, Spinner, Code

---

## Strengths

1. **Clean separation of concerns** - Well-organized module structure
2. **Type safety** - Extensive use of Rust enums for widget types and dock areas
3. **Full serialization support** - JSON import/export works reliably
4. **Sophisticated code generation** - Produces complete, compilable Rust/egui applications
5. **Grid snapping** - Configurable 1-64px grid alignment
6. **Docking system** - Widgets can be placed in 6 areas (Free, Top, Bottom, Left, Right, Center)
7. **Interactive preview** - Live widget manipulation with selection handles

---

## Identified Issues & Improvement Opportunities

### Critical Priority

#### 1. Large Monolithic File (`app.rs`)
**Problem:** At 1,668 lines, `app.rs` handles too many concerns:
- Application state
- Widget spawning
- Canvas rendering
- Widget drawing
- Inspector UI
- Palette UI
- Menu bar
- Code generation (544 lines alone!)

**Solution:** Split into focused modules:
```
src/
├── app/
│   ├── mod.rs           - RadBuilderApp struct and main update loop
│   ├── canvas.rs        - preview_panels_ui(), draw_widget(), draw_grid()
│   ├── palette.rs       - palette_ui(), palette_item()
│   ├── inspector.rs     - inspector_ui()
│   ├── menubar.rs       - top_bar()
│   └── codegen.rs       - generate_code() and all code emission logic
```

#### 2. No Automated Tests
**Problem:** Zero test coverage. The project relies entirely on manual testing.

**Solution:** Add test suites:
- Unit tests for `snap_pos_with_grid()`, `escape()`, widget defaults
- Integration tests for code generation (parse generated code, verify it compiles)
- Property-based tests for serialization roundtrips

#### 3. No CI/CD Pipeline
**Problem:** Only FUNDING.yml exists in `.github/workflows/`. No automated builds or tests.

**Solution:** Add GitHub Actions workflow:
```yaml
- cargo fmt --check
- cargo clippy
- cargo test
- cargo build --release
```

### High Priority

#### 4. Duplicated Widget Size Constants
**Problem:** Widget default sizes are defined twice:
- In `spawn_widget()` (lines 101-263)
- In ghost preview in `preview_panels_ui()` (lines 412-432)

**Solution:** Create a `WidgetKind::default_size()` method:
```rust
impl WidgetKind {
    pub fn default_size(&self) -> Vec2 {
        match self {
            WidgetKind::Label => vec2(140.0, 24.0),
            WidgetKind::Button => vec2(160.0, 32.0),
            // ...
        }
    }
}
```

#### 5. No Undo/Redo Support
**Problem:** Users cannot undo accidental deletions or modifications.

**Solution:** Implement command pattern with history stack:
```rust
enum Command {
    AddWidget(Widget),
    DeleteWidget(WidgetId),
    MoveWidget(WidgetId, Pos2, Pos2),
    ResizeWidget(WidgetId, Vec2, Vec2),
    ModifyProps(WidgetId, WidgetProps, WidgetProps),
}

struct History {
    undo_stack: Vec<Command>,
    redo_stack: Vec<Command>,
}
```

#### 6. No Native File Save/Load
**Problem:** Users must copy JSON from editor and paste it back in. No native file dialogs.

**Solution:** Add `rfd` (Rust File Dialog) crate for native save/load:
```rust
// File menu additions:
- Save Project (Ctrl+S)
- Save Project As...
- Open Project (Ctrl+O)
- Recent Projects submenu
```

#### 7. Missing Error Handling
**Problem:** JSON parsing failures are silently ignored:
```rust
// Current (line 1032):
if let Ok(p) = serde_json::from_str::<Project>(&self.generated) {
    self.project = p;
}
// User gets no feedback on failure
```

**Solution:** Add error state and display:
```rust
error_message: Option<String>,
// Display in UI when present
```

### Medium Priority

#### 8. Widget Registry/Plugin System
**Problem:** Adding new widgets requires modifying 4+ places in `app.rs`.

**Solution:** Create a widget registry:
```rust
trait WidgetFactory {
    fn kind(&self) -> WidgetKind;
    fn default_size(&self) -> Vec2;
    fn default_props(&self) -> WidgetProps;
    fn draw(&self, ui: &mut Ui, widget: &mut Widget);
    fn emit_code(&self, widget: &Widget, origin: &str) -> String;
}

struct WidgetRegistry {
    factories: HashMap<WidgetKind, Box<dyn WidgetFactory>>,
}
```

#### 9. Keyboard Shortcuts ✅ IMPLEMENTED
Basic shortcuts now available:
- `Delete` - Delete selected widget
- `Ctrl+C/V` - Copy/paste widget
- `Ctrl+D` - Duplicate widget
- `Ctrl+G` - Generate code

**Still needed:**
- `Ctrl+Z/Y` - Undo/redo
- `Ctrl+S` - Save project
- Arrow keys - Nudge selected widget

#### 10. Widget Alignment Tools
**Problem:** No way to align multiple widgets.

**Solution:** Add alignment toolbar when multiple widgets selected:
- Align left/center/right
- Align top/middle/bottom
- Distribute horizontally/vertically
- Match widths/heights

#### 11. Z-Order Controls
**Problem:** No way to change widget stacking order.

**Solution:** Add to inspector or context menu:
- Bring to Front
- Send to Back
- Bring Forward
- Send Backward

### Lower Priority (Future Enhancements)

#### 12. Multi-Page/Screen Support (from TODO)
Allow designing multiple screens/views that can be navigated between.

#### 13. Theming Support (from TODO)
- Font family, size, weight options
- Color customization per widget
- Dark/light theme preview
- Export theme as separate struct

#### 14. Additional Widgets (from TODO)
**Added:** TextArea, DragValue, Spinner, ColorPicker, Code

**Still needed:**
- Image widget (with image picker)
- Table/Grid widget
- Plot/Chart widget (egui_plot integration)
- Modal dialog / Window
- Tooltip
- Right-click context menu
- Tabs/TabBar
- Toolbar
- Statusbar
- Columns layout

#### 15. Live Preview Mode
Toggle between edit mode (current) and preview mode (interact with widgets without selection handles).

#### 16. Code Generation Improvements
- Generate idiomatic Rust (not string concatenation)
- Option to generate separate files (state.rs, ui.rs, main.rs)
- Generate event handlers as closures
- Add comments explaining generated code

#### 17. Project Templates
Starter templates:
- Settings dialog
- Login form
- Dashboard layout
- Wizard/multi-step form

---

## Suggested Roadmap

### Phase 1: Foundation (Code Quality)
1. Split `app.rs` into modules
2. Extract duplicated widget size constants
3. Add basic unit tests
4. Set up GitHub Actions CI

### Phase 2: Core UX Improvements
1. ~~Add keyboard shortcuts~~ ✅
2. ~~Add widget copy/paste~~ ✅
3. Implement undo/redo
4. Add native file save/load
5. Add error handling with user feedback

### Phase 3: Enhanced Editing
1. Multi-select and alignment tools
2. Z-order controls
3. Widget registry system
4. Live preview mode

### Phase 4: Expanded Features
1. Multi-page/screen support
2. Theming and styling options
3. Additional widget types
4. Improved code generation

### Phase 5: Polish
1. Project templates
2. Documentation and tutorials
3. Performance optimization
4. Accessibility improvements

---

## Technical Debt Notes

1. **Rust Edition 2024** in Cargo.toml appears non-standard (should be 2021)
2. `edit_mode` is stored in egui's temp data instead of app state (line 726-729)
3. Some `WidgetKind` arms in inspector use catch-all `_ => {}` that should be exhaustive
4. Generated code uses `from_id_source()` which is deprecated in favor of `from_id_salt()`
5. Tree widget parsing is duplicated between `draw_widget()` and `generate_code()`

---

## Development Tips

### Building
```bash
cargo build
cargo run
```

### Testing Generated Code
```bash
# Generate code in the tool, save to test_app/src/main.rs
cd test_app
cargo run
```

### Useful Commands
```bash
cargo fmt          # Format code
cargo clippy       # Lint
cargo doc --open   # Generate docs
```

---

*Last updated: 2025-12-26*
*Analysis performed by Claude*
