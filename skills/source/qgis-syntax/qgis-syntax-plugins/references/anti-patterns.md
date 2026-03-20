# qgis-syntax-plugins — Anti-Patterns

## AP-1: Incomplete unload() Method

### Wrong

```python
def initGui(self):
    self.action = QAction("My Plugin", self.iface.mainWindow())
    self.iface.addToolBarIcon(self.action)
    self.iface.addPluginToMenu("&My Plugin", self.action)
    self.dock = QDockWidget("Panel", self.iface.mainWindow())
    self.iface.addDockWidget(Qt.RightDockWidgetArea, self.dock)

def unload(self):
    self.iface.removePluginMenu("&My Plugin", self.action)
    # MISSING: removeToolBarIcon and removeDockWidget
```

### Why It Fails

Ghost toolbar icons and dock widgets persist after plugin deactivation. Users see duplicate UI elements each time they re-enable the plugin.

### Correct

```python
def unload(self):
    self.iface.removePluginMenu("&My Plugin", self.action)
    self.iface.removeToolBarIcon(self.action)
    self.iface.removeDockWidget(self.dock)
    del self.dock
```

**Rule**: ALWAYS remove every GUI element that `initGui()` creates. Keep a list of all actions and widgets.

---

## AP-2: Creating GUI Elements in __init__

### Wrong

```python
def __init__(self, iface):
    self.iface = iface
    self.action = QAction("My Plugin", self.iface.mainWindow())  # WRONG
    self.iface.addToolBarIcon(self.action)  # WRONG
```

### Why It Fails

`__init__` is called during QGIS startup for all installed plugins, even disabled ones. Creating GUI elements here adds UI for plugins the user has not enabled.

### Correct

```python
def __init__(self, iface):
    self.iface = iface
    # Store reference only — no GUI creation

def initGui(self):
    self.action = QAction("My Plugin", self.iface.mainWindow())
    self.iface.addToolBarIcon(self.action)
```

**Rule**: NEVER create GUI elements in `__init__`. ALWAYS use `initGui()`.

---

## AP-3: Accessing GUI from Background Thread

### Wrong

```python
from qgis.core import QgsTask, QgsApplication

class MyTask(QgsTask):
    def __init__(self, iface):
        super().__init__("My Task")
        self.iface = iface

    def run(self):
        # WRONG — accessing iface from background thread
        layer = self.iface.activeLayer()
        self.iface.messageBar().pushMessage("Done", "Result", level=Qgis.Info)
        return True
```

### Why It Fails

Qt GUI objects are NOT thread-safe. Accessing `iface`, `QgsProject.instance()`, or any widget from a background thread causes segfaults, deadlocks, or silent data corruption.

### Correct

```python
class MyTask(QgsTask):
    def __init__(self, data):
        super().__init__("My Task")
        self.data = data  # Pass copies, not live references
        self.result_data = None

    def run(self):
        # Process data only — no GUI access
        self.result_data = self.process(self.data)
        return True

    def finished(self, result):
        # finished() runs on the main thread — GUI access is safe here
        if result:
            iface.messageBar().pushMessage(
                "Done", "Processing complete", level=Qgis.Success
            )
```

**Rule**: NEVER access `iface` or any GUI object in `QgsTask.run()`. Use `finished()` for GUI updates.

---

## AP-4: Missing objectName on QActions

### Wrong

```python
def initGui(self):
    self.action = QAction("My Plugin", self.iface.mainWindow())
    # No objectName set
```

### Why It Fails

QGIS uses `objectName` to persist toolbar customizations. Without it, toolbar positions reset on restart. Multiple plugins with unnamed actions cause conflicts.

### Correct

```python
def initGui(self):
    self.action = QAction("My Plugin", self.iface.mainWindow())
    self.action.setObjectName("myPluginMainAction")
```

**Rule**: ALWAYS call `setObjectName()` with a unique string on every QAction.

---

## AP-5: Raising Exceptions in QgsTask.run()

### Wrong

```python
class MyTask(QgsTask):
    def run(self):
        data = self.load_data()
        if data is None:
            raise ValueError("No data found")  # WRONG
        return True
```

### Why It Fails

Unhandled exceptions in `QgsTask.run()` crash QGIS or cause the task to hang indefinitely without calling `finished()`.

### Correct

```python
class MyTask(QgsTask):
    def run(self):
        try:
            data = self.load_data()
            if data is None:
                self.error_msg = "No data found"
                return False
            return True
        except Exception as e:
            self.error_msg = str(e)
            return False

    def finished(self, result):
        if not result:
            QgsMessageLog.logMessage(
                self.error_msg, "My Plugin", Qgis.Critical
            )
```

**Rule**: NEVER raise exceptions in `QgsTask.run()`. Return `False` and log the error.

---

## AP-6: Committing Compiled Files to Version Control

### Wrong

```
my_plugin/
├── resources.py      # COMPILED — should not be in repo
├── resources.qrc     # Source — OK
├── ui_dialog.py      # COMPILED — should not be in repo
├── dialog.ui         # Source — OK
```

### Why It Fails

Compiled files are platform-specific and QGIS-version-specific. They cause merge conflicts and mask the actual source files. Different PyQt5 versions generate different output.

### Correct

Add to `.gitignore`:
```
resources.py
resources_rc.py
ui_*.py
```

Build during deployment:
```bash
pyrcc5 -o resources.py resources.qrc
pyuic5 -o ui_dialog.py dialog.ui
```

**Rule**: NEVER commit compiled resource or UI files. ALWAYS regenerate during build.

---

## AP-7: Forgetting to Disconnect Signals

### Wrong

```python
def initGui(self):
    self.iface.currentLayerChanged.connect(self.on_layer_changed)

def unload(self):
    # MISSING: disconnect signal
    pass
```

### Why It Fails

After plugin deactivation, the signal still fires and calls `self.on_layer_changed` on a partially destroyed object. This causes `RuntimeError: wrapped C/C++ object has been deleted`.

### Correct

```python
def initGui(self):
    self.iface.currentLayerChanged.connect(self.on_layer_changed)

def unload(self):
    self.iface.currentLayerChanged.disconnect(self.on_layer_changed)
```

**Rule**: ALWAYS disconnect every signal connection in `unload()`.

---

## AP-8: Using Wrong Parent for Dialogs

### Wrong

```python
def run(self):
    dlg = QDialog()  # No parent — floats behind main window
    dlg.exec_()
```

### Why It Fails

Dialogs without a parent window float independently, can get lost behind the main window, and are not properly garbage-collected by Qt.

### Correct

```python
def run(self):
    dlg = QDialog(self.iface.mainWindow())
    dlg.exec_()
```

**Rule**: ALWAYS pass `self.iface.mainWindow()` as parent for dialogs and QActions.

---

## AP-9: Plugin Folder Name with Invalid Characters

### Wrong

```
My Plugin v2.0/
├── __init__.py
├── metadata.txt
```

### Why It Fails

QGIS Plugin Repository rejects folder names with spaces, special characters, or names starting with digits. Python cannot import modules with spaces in the name.

### Correct

```
my_plugin_v2/
├── __init__.py
├── metadata.txt
```

**Rule**: Plugin folder names MUST contain only ASCII letters (A-Z, a-z), digits (0-9), underscores, and hyphens. NEVER start with a digit.

---

## AP-10: Unprefixed Settings Keys

### Wrong

```python
self.settings.setValue("lastDirectory", "/home/user/data")
```

### Why It Fails

Settings are global. Without a plugin-specific prefix, settings collide with other plugins or QGIS core settings.

### Correct

```python
self.settings.setValue("MyPlugin/lastDirectory", "/home/user/data")
```

**Rule**: ALWAYS prefix QgsSettings keys with the plugin name.
