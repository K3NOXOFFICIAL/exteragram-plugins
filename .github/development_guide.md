# Exteragram Plugin Development Guide for GitHub Copilot

## Table of Contents
1. [Plugin Overview](#plugin-overview)
2. [Essential Plugin Structure](#essential-plugin-structure)
3. [Required Metadata](#required-metadata)
4. [Base Class and Lifecycle](#base-class-and-lifecycle)
5. [Hook System Patterns](#hook-system-patterns)
6. [Settings and UI](#settings-and-ui)
7. [Common Utility Patterns](#common-utility-patterns)
8. [Error Handling Best Practices](#error-handling-best-practices)
9. [Complete Plugin Templates](#complete-plugin-templates)
10. [Real-World Examples](#real-world-examples)

---

## Plugin Overview

Exteragram is a Telegram client with a plugin system that allows developers to extend functionality through Python plugins. Plugins can intercept messages, modify UI elements, add menu items, hook into Telegram API calls, and interact with Android UI components.

**Key Concepts:**
- Plugins are Python files with `.plugin` extension
- All plugins inherit from `BasePlugin`
- Plugin system supports hooks for message processing, API requests, and updates
- Rich settings UI system with various controls
- Integration with Android UI components and Telegram client

---

## Essential Plugin Structure

### Basic Template
```python
__id__ = "my_plugin_id"
__name__ = "My Plugin Name"
__description__ = "Plugin description with functionality details"
__author__ = "Your Name or @username"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"

from base_plugin import BasePlugin

class MyPluginPlugin(BasePlugin):
    def on_plugin_load(self):
        # Initialize your plugin
        self.log("My plugin loaded successfully!")
        
        # Register hooks here
        # self.add_on_send_message_hook()
        # self.add_hook("TL_messages_sendMessage")
    
    def on_plugin_unload(self):
        # Cleanup when plugin is disabled
        self.log("My plugin unloaded!")
```

### File Organization
- Main plugin file: `[plugin_name].plugin`
- Settings are automatically handled by the framework
- File operations use built-in `file_utils` utilities
- Cache and temp files stored in appropriate directories

---

## Required Metadata

All plugins must define these metadata fields as plain strings:

```python
# Required metadata
__id__ = "unique_plugin_identifier"  # 2-32 chars, letters/numbers/_/-, starts with letter
__name__ = "Human Readable Name"     # Display name in plugin list

# Recommended metadata
__description__ = "What this plugin does and how to use it"
__author__ = "Your Name or @TelegramUsername"
__version__ = "1.0.0"               # Semantic version, defaults to "1.0" if missing
__icon__ = "stickerPack/index"       # Sticker for plugin icon (starts from 0)
__min_version__ = "11.12.0"          # Minimum Exteragram version required
```

**Metadata Rules:**
- Use plain strings only (no concatenation, formatting)
- `__id__` and `__name__` are required
- Version is validated if `__min_version__` is present
- Icons use format: `stickerPackName/stickerIndex`
- Author can be plain text or Telegram username (@username)

---

## Base Class and Lifecycle

### Essential Methods
```python
class MyPluginPlugin(BasePlugin):
    def on_plugin_load(self):
        """Called when plugin is enabled or app starts"""
        # Register hooks
        # Initialize data
        # Set up UI elements
        pass
    
    def on_plugin_unload(self):
        """Called when plugin is disabled or app closes"""
        # Cleanup hooks
        # Save data
        # Dismiss dialogs
        pass
```

### Optional Lifecycle Methods
```python
def on_app_event(self, event_type):
    """Called for app lifecycle events"""
    from base_plugin import AppEvent
    if event_type == AppEvent.START:
        pass
    elif event_type == AppEvent.STOP:
        pass
    elif event_type == AppEvent.PAUSE:
        pass
    elif event_type == AppEvent.RESUME:
        pass

def create_settings(self):
    """Return list of UI components for settings screen"""
    from ui.settings import Switch, Input, Header, Divider
    return [
        Header(text="Plugin Settings"),
        Switch(key="enable_feature", text="Enable Feature", default=True),
        Input(key="api_key", text="API Key", default=""),
    ]
```

### Settings Management
```python
# Get settings values
enabled = self.get_setting("enable_feature", False)
api_key = self.get_setting("api_key", "")

# Set settings values programmatically
self.set_setting("enable_feature", True)

# Import/export settings
all_settings = self.export_settings()
self.import_settings({"enable_feature": True})

# Force settings UI reload
self.set_setting("main_option", "A", reload_settings=True)
```

---

## Hook System Patterns

### 1. Message Hooks
```python
class WeatherPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
    
    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        from base_plugin import HookResult, HookStrategy
        
        # Check if message matches our command
        if not isinstance(params.message, str) or not params.message.startswith(".wt"):
            return HookResult()
        
        try:
            # Process the command
            parts = params.message.strip().split(" ", 1)
            city = parts[1].strip() if len(parts) > 1 else "Moscow"
            
            # Example: fetch weather data
            weather_data = self.fetch_weather(city)
            
            # Modify the message
            params.message = f"Weather in {city}: {weather_data}"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
        except Exception as e:
            self.log(f"Weather error: {str(e)}")
            params.message = f"Error: {str(e)}"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

### 2. API Request Hooks
```python
class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        # Hook specific API requests
        self.add_hook("TL_messages_setTyping")
        self.add_hook("TL_account_updateStatus")
    
    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        from base_plugin import HookResult, HookStrategy
        
        # Block typing status
        if request_name == "TL_messages_setTyping":
            if self.get_setting("block_typing", True):
                return HookResult(strategy=HookStrategy.CANCEL)
        
        # Force offline status
        if request_name == "TL_account_updateStatus":
            if self.get_setting("force_offline", True):
                return HookResult(strategy=HookStrategy.CANCEL)
        
        return HookResult()
    
    def post_request_hook(self, request_name: str, account: int, request: Any, response: Any, error: Any) -> HookResult:
        # Handle responses
        return HookResult()
```

### 3. Update Hooks
```python
class MessageFilterPlugin(BasePlugin):
    def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
        from base_plugin import HookResult, HookStrategy
        
        if update_name == "TL_updateNewMessage":
            # Process incoming messages
            if hasattr(update, 'message') and hasattr(update.message, 'message'):
                message_text = update.message.message
                if "secret" in message_text:
                    # Modify the message
                    update.message.message = "[FILTERED]"
                    return HookResult(strategy=HookStrategy.MODIFY, update=update)
        
        return HookResult()
```

### HookResult Strategies
```python
HookResult()                          # No modification (DEFAULT)
HookResult(strategy=HookStrategy.CANCEL)  # Cancel the operation
HookResult(strategy=HookStrategy.MODIFY, params=modified_params)  # Modify parameters
```

---

## Settings and UI

### Available UI Components
```python
from ui.settings import (
    Header,           # Section header
    Divider,          # Visual separator
    Switch,           # Boolean toggle
    Selector,         # Dropdown with options
    Input,            # Text input
    EditText,         # Multi-line text input
    Text              # Clickable text item
)

def create_settings(self):
    return [
        Header(text="General Settings"),
        Switch(
            key="enable_feature",
            text="Enable Feature",
            default=True,
            subtext="Turn on this awesome feature",
            icon="msg_settings",
            on_change=self.on_setting_changed
        ),
        Selector(
            key="theme",
            text="Theme",
            default=0,
            items=["Light", "Dark", "Auto"],
            on_change=self.on_theme_changed
        ),
        Input(
            key="api_key",
            text="API Key",
            default="",
            subtext="Enter your API key here",
            icon="msg_text"
        ),
        Divider(),
        Header(text="Advanced"),
        Text(
            text="Reset to Defaults",
            icon="msg_info",
            red=True,
            on_click=self.reset_settings
        )
    ]
```

### Menu Items
```python
from base_plugin import MenuItemData, MenuItemType

def on_plugin_load(self):
    # Add menu items
    self.add_menu_item(
        MenuItemData(
            menu_type=MenuItemType.MESSAGE_CONTEXT_MENU,
            text="Plugin Action",
            on_click=self.handle_menu_click,
            icon="msg_info",
            item_id="my_menu_item"
        )
    )

def handle_menu_click(self, context):
    # Access context data
    message = context.get("message")
    user = context.get("user")
    chat = context.get("chat")
    
    # Perform action
    self.log(f"Menu item clicked for {user.first_name if user else 'Unknown'}")
```

### Dialogs and Notifications
```python
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper
from client_utils import get_last_fragment

# Show bulletin notification
BulletinHelper.show_info("Operation completed successfully!")

# Show error notification
BulletinHelper.show_error("Something went wrong!")

# Show confirmation dialog
def show_confirmation(self):
    current_fragment = get_last_fragment()
    if not current_fragment:
        return
    
    activity = current_fragment.getParentActivity()
    if not activity:
        return
    
    builder = AlertDialogBuilder(activity)
    builder.set_title("Confirm Action")
    builder.set_message("Are you sure you want to proceed?")
    
    def on_positive_click(bld, which):
        self.perform_action()
        bld.dismiss()
    
    def on_negative_click(bld, which):
        bld.dismiss()
    
    builder.set_positive_button("Yes", on_positive_click)
    builder.set_negative_button("No", on_negative_click)
    builder.show()
```

---

## Common Utility Patterns

### 1. Background Operations
```python
from client_utils import run_on_queue, run_on_ui_thread
from android_utils import log

def process_data_async(self, data):
    """Run long-running operation on background thread"""
    def _process():
        # Perform heavy work here
        result = self.fetch_from_api(data)
        
        # Update UI on main thread
        run_on_ui_thread(lambda: self.update_ui(result))
    
    run_on_queue(_process)
```

### 2. File Operations
```python
from file_utils import get_plugins_dir, write_file, read_file, ensure_dir_exists
import os

def save_data(self):
    """Save plugin data to file"""
    data_dir = os.path.join(get_plugins_dir(), "my_plugin")
    ensure_dir_exists(data_dir)
    
    data = {"key": "value", "timestamp": time.time()}
    file_path = os.path.join(data_dir, "data.json")
    write_file(file_path, json.dumps(data))

def load_data(self):
    """Load plugin data from file"""
    file_path = os.path.join(get_plugins_dir(), "my_plugin", "data.json")
    try:
        content = read_file(file_path)
        return json.loads(content)
    except:
        return {}
```

### 3. API Integration
```python
import requests
from client_utils import send_message

def fetch_weather(self, city):
    """Example API integration"""
    try:
        url = f"https://api.weather.com/v1/current?q={city}"
        response = requests.get(url, timeout=10)
        if response.status_code == 200:
            data = response.json()
            return self.format_weather(data)
        else:
            return "Weather service unavailable"
    except Exception as e:
        self.log(f"API error: {str(e)}")
        return "Network error"

def send_message_to_user(self, user_id, text):
    """Send message programmatically"""
    from client_utils import send_text
    send_text(user_id, text)
```

### 4. Markdown Support
```python
from markdown_utils import parse_markdown

def send_formatted_message(self, peer_id, markdown_text):
    """Send message with markdown formatting"""
    try:
        parsed = parse_markdown(markdown_text)
        
        params = {
            "peer": peer_id,
            "message": parsed.text,
            "entities": []
        }
        
        for raw_entity in parsed.entities:
            tlrpc_entity = raw_entity.to_tlrpc_object()
            params["entities"].append(tlrpc_entity)
        
        send_message(params)
        
    except Exception as e:
        self.log(f"Markdown parsing error: {str(e)}")
        # Fallback to plain text
        send_text(peer_id, markdown_text)
```

---

## Error Handling Best Practices

### 1. Comprehensive Error Handling
```python
def on_send_message_hook(self, account, params):
    try:
        # Your plugin logic here
        result = self.process_message(params.message)
        params.message = result
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
    except requests.RequestException as e:
        # Network-related errors
        self.log(f"Network error: {str(e)}")
        params.message = "Network error. Please try again later."
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
    except ValueError as e:
        # Data parsing errors
        self.log(f"Data parsing error: {str(e)}")
        params.message = "Invalid data format."
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
    except Exception as e:
        # Catch-all for unexpected errors
        self.log(f"Unexpected error: {str(e)}")
        self.log(traceback.format_exc())  # Log full traceback for debugging
        
        # Don't crash, provide graceful fallback
        params.message = f"An error occurred: {str(e)}"
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

### 2. Thread-Safe UI Updates
```python
def update_ui_safely(self):
    """Ensure UI updates happen on main thread"""
    try:
        run_on_ui_thread(self._perform_ui_update)
    except Exception as e:
        self.log(f"UI update error: {str(e)}")

def _perform_ui_update(self):
    try:
        # Actual UI update code
        self.update_progress_bar(value)
        self.set_status_text("Completed")
    except Exception as e:
        self.log(f"UI update failed: {str(e)}")
```

### 3. Resource Management
```python
class MyPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.dialog = None
        self.temp_files = []
    
    def on_plugin_load(self):
        # Initialize resources
        pass
    
    def on_plugin_unload(self):
        # Clean up resources
        if self.dialog:
            try:
                self.dialog.dismiss()
            except:
                pass
        
        # Clean up temp files
        for file_path in self.temp_files:
            try:
                os.remove(file_path)
            except:
                pass
        
        self.log("Plugin cleanup completed")
```

---

## Complete Plugin Templates

### 1. Simple Command Plugin
```python
__id__ = "my_simple_command"
__name__ = "Simple Command Plugin"
__description__ = "Responds to .hello command"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.bulletin import BulletinHelper

class SimpleCommandPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
        self.log("Simple command plugin loaded!")
    
    def on_send_message_hook(self, account: int, params) -> HookResult:
        if not isinstance(params.message, str):
            return HookResult()
        
        if params.message.strip() == ".hello":
            params.message = "Hello! This is a simple plugin response."
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
        return HookResult()
```

### 2. API Integration Plugin
```python
__id__ = "weather_plugin"
__name__ = "Weather Plugin"
__description__ = "Get weather information with .weather command"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"

import requests
from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import run_on_queue, run_on_ui_thread
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper

class WeatherPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
    
    def on_send_message_hook(self, account: int, params) -> HookResult:
        if not isinstance(params.message, str) or not params.message.startswith(".weather"):
            return HookResult()
        
        try:
            # Parse command
            parts = params.message.split(" ", 1)
            city = parts[1].strip() if len(parts) > 1 else "Moscow"
            
            if not city:
                params.message = "Usage: .weather <city>"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            # Show loading
            self.show_loading_dialog()
            
            # Fetch weather in background
            run_on_queue(lambda: self.fetch_weather_async(city, params))
            
            return HookResult(strategy=HookStrategy.CANCEL)  # Cancel original message
            
        except Exception as e:
            self.log(f"Weather plugin error: {str(e)}")
            params.message = f"Error: {str(e)}"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    def fetch_weather_async(self, city, original_params):
        try:
            # Simulate API call
            response = requests.get(f"https://api.weather.com/v1/current?q={city}", timeout=10)
            
            if response.status_code == 200:
                weather_data = response.json()
                formatted_weather = self.format_weather(weather_data, city)
            else:
                formatted_weather = f"Could not fetch weather for {city}"
            
            # Send result on UI thread
            run_on_ui_thread(lambda: self.send_weather_result(formatted_weather, original_params))
            
        except Exception as e:
            run_on_ui_thread(lambda: self.send_error_message(str(e), original_params))
    
    def format_weather(self, data, city):
        # Format weather data for display
        temp = data.get('temperature', 'N/A')
        condition = data.get('condition', 'Unknown')
        return f"Weather in {city}: {temp}°C, {condition}"
    
    def show_loading_dialog(self):
        from client_utils import get_last_fragment
        fragment = get_last_fragment()
        if fragment:
            activity = fragment.getParentActivity()
            if activity:
                self.dialog = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_SPINNER)
                self.dialog.set_title("Fetching Weather...")
                self.dialog.set_cancelable(False)
                self.dialog.show()
    
    def send_weather_result(self, weather_text, original_params):
        if self.dialog:
            self.dialog.dismiss()
        
        from client_utils import send_text
        send_text(original_params.peer, weather_text)
        
        BulletinHelper.show_success("Weather fetched successfully!")
    
    def send_error_message(self, error_msg, original_params):
        if self.dialog:
            self.dialog.dismiss()
        
        from client_utils import send_text
        send_text(original_params.peer, f"Weather error: {error_msg}")
```

### 3. Settings-Rich Plugin
```python
__id__ = "text_enhancer"
__name__ = "Text Enhancer"
__description__ = "Enhances text with custom formatting"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Switch, Input, Selector, Divider
from client_utils import run_on_queue

class TextEnhancerPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
    
    def create_settings(self):
        return [
            Header(text="Enhancement Settings"),
            Switch(
                key="enable_enhancement",
                text="Enable Text Enhancement",
                default=True
            ),
            Switch(
                key="add_emojis",
                text="Add Emojis",
                default=True
            ),
            Selector(
                key="enhancement_style",
                text="Enhancement Style",
                default=0,
                items=["Simple", "Fancy", "Minimal"]
            ),
            Divider(),
            Header(text="Custom Settings"),
            Input(
                key="custom_prefix",
                text="Custom Prefix",
                default=""
            )
        ]
    
    def on_send_message_hook(self, account: int, params) -> HookResult:
        if not self.get_setting("enable_enhancement", True):
            return HookResult()
        
        if not isinstance(params.message, str):
            return HookResult()
        
        try:
            original_text = params.message
            
            # Apply enhancements based on settings
            enhanced_text = self.enhance_text(original_text)
            
            params.message = enhanced_text
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
        except Exception as e:
            self.log(f"Text enhancement error: {str(e)}")
            return HookResult()  # Return original message on error
    
    def enhance_text(self, text):
        """Apply text enhancements based on settings"""
        add_emojis = self.get_setting("add_emojis", True)
        style = self.get_setting("enhancement_style", 0)
        prefix = self.get_setting("custom_prefix", "")
        
        # Apply style
        if style == 1:  # Fancy
            text = f"✨ {text} ✨"
        elif style == 2:  # Minimal
            text = f"[{text}]"
        
        # Add emojis if enabled
        if add_emojis:
            text += " 😊"
        
        # Add custom prefix
        if prefix:
            text = f"{prefix} {text}"
        
        return text
```

---

## Real-World Examples from Community Plugins

### 1. Message Processing Pattern (from AutoSignaturePlugin)
```python
def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    # Check if we have content to process
    message_content = getattr(params, 'message', None) or getattr(params, 'caption', None)
    if not message_content or not isinstance(message_content, str):
        return HookResult()
    
    # Skip if it's a command
    if message_content.startswith('.'):
        return HookResult()
    
    # Get signature configuration
    signature_text = self.get_setting("signature_text", "")
    if not signature_text:
        return HookResult()
    
    # Apply signature based on settings
    if self.get_setting("signature_placement", "end") == "beginning":
        params.message = f"{signature_text}\n{message_content}"
    else:
        params.message = f"{message_content}\n{signature_text}"
    
    return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

### 2. Caching Pattern (from AccountAgePlugin)
```python
class CacheManager:
    def __init__(self, plugin):
        self.plugin = plugin
        self.cache = {}
    
    def get_cached_result(self, user_id, method):
        cache_key = f"{user_id}_{method}"
        if cache_key in self.cache:
            entry = self.cache[cache_key]
            if time.time() - entry['timestamp'] < self.get_cache_duration():
                return entry['result']
        return None
    
    def set_cached_result(self, user_id, method, result):
        cache_key = f"{user_id}_{method}"
        self.cache[cache_key] = {
            'result': result,
            'timestamp': time.time()
        }
```

### 3. Multi-language Pattern (from QuickSettings)
```python
class Locales:
    EN = {
        "settings_title": "Quick Settings",
        "menu_item": "Plugin Settings",
        "no_plugins": "No plugins with settings found"
    }
    RU = {
        "settings_title": "Быстрые Настройки",
        "menu_item": "Настройки Плагинов",
        "no_plugins": "Плагинов с настройками не найдено"
    }

def localise(key):
    locale = LocaleController.getInstance().getCurrentLocaleInfo()
    lang = locale.langCode
    return getattr(Locales, lang, Locales.EN).get(key, key)
```

### 4. Background Worker Pattern (from AIStatusPlugin)
```python
class AIStatusPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.running = False
        self.worker_thread = None
    
    def on_plugin_load(self):
        self.running = True
        self.worker_thread = threading.Thread(target=self.worker_loop, daemon=True)
        self.worker_thread.start()
    
    def worker_loop(self):
        last_updates = {}
        while self.running:
            try:
                current_time = time.time()
                
                # Check each element type
                for element_type in ["name", "bio", "username"]:
                    if self.should_update(element_type, current_time, last_updates):
                        self.update_element(element_type)
                        last_updates[element_type] = current_time
                
                time.sleep(30)  # Check every 30 seconds
                
            except Exception as e:
                self.log(f"Worker loop error: {str(e)}")
                time.sleep(60)  # Wait longer on error
    
    def on_plugin_unload(self):
        self.running = False
        if self.worker_thread:
            self.worker_thread.join(timeout=5)
```

---

## Performance Optimization Tips

### 1. Efficient Hook Registration
```python
# Good: Register specific hooks only
self.add_hook("TL_messages_sendMessage")

# Avoid: Registering too many generic hooks
# self.add_hook("*")  # This would catch everything and slow down the app
```

### 2. Caching Expensive Operations
```python
from functools import lru_cache
import time

class ExpensiveAPIClient:
    def __init__(self):
        self._cache = {}
        self._cache_duration = 3600  # 1 hour
    
    @lru_cache(maxsize=100)
    def get_cached_data(self, query_hash):
        return self.fetch_data(query_hash)
    
    def get_data_with_cache(self, query):
        cache_key = hashlib.md5(query.encode()).hexdigest()
        cache_entry = self._cache.get(cache_key)
        
        if cache_entry and time.time() - cache_entry['timestamp'] < self._cache_duration:
            return cache_entry['data']
        
        data = self.fetch_data(query)
        self._cache[cache_key] = {'data': data, 'timestamp': time.time()}
        return data
```

### 3. Memory Management
```python
class MemoryEfficientPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        # Use generators for large datasets
        self.data_generator = None
        self.temp_files = []
    
    def process_large_dataset(self):
        # Process in chunks instead of loading everything
        for chunk in self.yield_data_chunks():
            yield self.process_chunk(chunk)
    
    def yield_data_chunks(self):
        # Generator that yields data in manageable chunks
        chunk_size = 1000
        current_chunk = []
        
        for item in self.get_all_items():
            current_chunk.append(item)
            if len(current_chunk) >= chunk_size:
                yield current_chunk
                current_chunk = []
        
        if current_chunk:
            yield current_chunk
    
    def on_plugin_unload(self):
        # Clean up generators and temp files
        self.data_generator = None
        
        for temp_file in self.temp_files:
            try:
                os.remove(temp_file)
            except:
                pass
```

---

## Testing and Debugging

### 1. Debug Logging
```python
def debug_log(self, message, level="INFO"):
    if self.get_setting("debug_mode", False):
        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
        self.log(f"[{timestamp}] [{level}] {message}")

def test_plugin_functionality(self):
    """Test plugin functions during development"""
    try:
        # Test your main functionality
        result = self.test_message_processing("test message")
        debug_log(f"Test result: {result}")
        
        # Test settings
        debug_log(f"Plugin enabled: {self.get_setting('enable_plugin', False)}")
        
        return True
    except Exception as e:
        debug_log(f"Test failed: {str(e)}", "ERROR")
        return False
```

### 2. Error Recovery Patterns
```python
def safe_execute(self, func, *args, **kwargs):
    """Wrapper for safe function execution"""
    try:
        return func(*args, **kwargs)
    except requests.RequestException as e:
        self.handle_network_error(e)
        return None
    except ValueError as e:
        self.handle_parsing_error(e)
        return None
    except Exception as e:
        self.handle_unexpected_error(e)
        return None

def handle_network_error(self, error):
    self.log(f"Network error: {str(error)}")
    BulletinHelper.show_error("Network connection failed")

def handle_parsing_error(self, error):
    self.log(f"Parsing error: {str(error)}")
    BulletinHelper.show_error("Invalid data format")

def handle_unexpected_error(self, error):
    self.log(f"Unexpected error: {str(error)}")
    BulletinHelper.show_error("An unexpected error occurred")
```

---

## Development Setup Guide

### Getting Started with Plugin Development

#### 1. Download exteraGram
Make sure you're using the latest version of exteraGram or derivative client.
- Download from the [beta channel](https://t.me/exteraGramCI)

#### 2. Enable Plugins Engine
After logging into your account:
- Go to `exteraGram Preferences` > `Plugins`
- Enable plugins engine
- Tap info button and enable developer mode

#### 3. Bootstrap Project
Create a folder on your PC and create a Python file (e.g., `first_plugin.py`).
- Create virtual environment
- Install `exteragram-utils` for typings and hot-reload client

#### 4. Connecting to Phone
Connect your phone to your PC using cable. Ensure [ADB](https://developer.android.com/tools/adb) is on your `PATH`.

```bash
# Replace first_plugin.py with your actual filename
extera first_plugin.py

# For debugging support
extera first_plugin.py --debug
```

#### VS Code Remote Debugging Example:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python Debugger: Remote Attach exteraGram",
            "type": "debugpy",
            "request": "attach",
            "connect": {
                "host": "localhost",
                "port": 5678
            },
            "pathMappings": [
                {
                    "localRoot": "/Users/alexeyzavar/Projects/extera-plugins/first_plugin.py",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins/first_plugin.py"
                }
            ]
        }
    ]
}
```

**Note**: `remoteRoot` should end with `PLUGIN_ID.py`.

### Available Pre-installed Libraries

The plugin environment comes with these pre-installed libraries:

#### Python Version
- Python: `3.11`

#### Pre-installed Pip Packages
- `beautifulsoup4`: HTML and XML parsing for web scraping
- `debugpy`: Microsoft's Python debugger for remote debugging
- `lxml`: Powerful XML/HTML processing library
- `packaging`: Python package utilities
- `pillow`: Python Imaging Library (PIL fork)
- `requests`: HTTP library for making web requests
- `PyYAML`: YAML parser and emitter

**Important**: If your plugin requires additional libraries not on this list, you must implement the functionality yourself or find Java alternatives. The plugin system doesn't support runtime package installation.

---

## Complete API Reference

### Plugin Metadata Requirements

All plugins must define these metadata fields:

| Field           | Description                                                                                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `__id__`        | **Required.** A unique identifier for the plugin. Must be 2-32 characters long, start with a letter, and contain only letters, numbers, dashes (`-`), and underscores (`_`). |
| `__name__`      | **Required.** The human-readable name of the plugin, displayed in the UI.                                                                                                |
| `__description__` | A description of the plugin's functionality. Supports basic Markdown.                                                                                                    |
| `__author__`    | The name of the plugin author. Can be a plain name or a Telegram username (e.g., `@username`).                                                                           |
| `__version__`   | The version of the plugin. Defaults to `1.0`.                                                                                                                            |
| `__icon__`      | The icon for the plugin, in the format `stickerPackName/stickerIndex`.                                                                                                   |
| `__min_version__` | The minimum version of exteraGram required for the plugin to function correctly.                                                                                        |

### Base Plugin Class Methods

#### Lifecycle Methods
- `on_plugin_load(self)`: Called when the plugin is enabled or the app starts. Use this to register hooks, initialize resources, and set up UI elements.
- `on_plugin_unload(self)`: Called when the plugin is disabled or the app closes. Use this to unregister hooks, clean up resources, and save data.
- `on_app_event(self, event_type: AppEvent)`: Called for app lifecycle events. `event_type` can be `AppEvent.START`, `AppEvent.STOP`, `AppEvent.PAUSE`, or `AppEvent.RESUME`.

#### Settings Methods
- `create_settings(self) -> List[Any]`: Returns a list of UI components for the plugin's settings page.
- `get_setting(self, key: str, default: Any) -> Any`: Retrieves a setting value.
- `set_setting(self, key: str, value: Any, reload_settings: bool = False)`: Sets a setting value.
- `export_settings(self) -> Dict[str, Any]`: Exports all of the plugin's settings to a dictionary.
- `import_settings(self, settings: Dict[str, Any])`: Imports settings from a dictionary.

#### Hook Methods
- `add_on_send_message_hook(self)`: Registers the `on_send_message_hook` method.
- `add_hook(self, request_name: str)`: Registers a hook for a specific API request.
- `hook_method(self, target_method, hook_handler, priority: int = 0) -> object`: Hooks a Java method using Xposed. Returns an "unhook" object.
- `unhook_method(self, unhook_obj: object)`: Removes a previously applied Xposed hook.

#### Menu Item Methods
- `add_menu_item(self, menu_item_data: MenuItemData) -> MenuItemData`: Adds a menu item to the UI.
- `remove_menu_item(self, menu_item_data: MenuItemData)`: Removes a menu item.

### Hook Result System

#### `HookResult` Class
Determines the outcome of a hook with these properties:
- `strategy`: The `HookStrategy` to use
- `params`: The modified parameters for a `MODIFY` strategy
- `update`: The modified update for a `MODIFY` strategy  
- `request`: The modified request for a `MODIFY` strategy
- `response`: The modified response for a `MODIFY` strategy

#### `HookStrategy` Enum
- `DEFAULT`: The operation proceeds without modification
- `CANCEL`: The operation is canceled
- `MODIFY`: The parameters of the operation are modified
- `MODIFY_FINAL`: Same as `MODIFY`, but no other plugins' hooks for this event will be called

### Menu System

#### `MenuItemData` Class
Represents a custom menu item with these properties:
- `menu_type: MenuItemType`: The menu to add the item to
- `text: str`: The text of the menu item
- `on_click: Callable[[Dict[str, Any]], None]`: The function to call when the item is clicked
- `item_id: str`: A unique ID for the item
- `icon: str`: The name of a drawable resource to use as an icon
- `subtext: str`: Additional text to display below the main text
- `condition: str`: A MVEL expression to conditionally show the item
- `priority: int`: The priority of the item in the menu

#### `MenuItemType` Enum
- `MESSAGE_CONTEXT_MENU`: The menu that appears when pressing a message
- `DRAWER_MENU`: The main navigation drawer menu
- `CHAT_ACTION_MENU`: The three-dot menu inside a chat screen
- `PROFILE_ACTION_MENU`: The three-dot menu on a user, bot, or channel profile screen

### Client Utils Module

| Function/Class          | Description                                                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `run_on_queue(callable, queue, delay)` | Executes a function on a background thread. `queue` can be one of `STAGE_QUEUE`, `GLOBAL_QUEUE`, etc. `delay` is in milliseconds. |
| `send_request(request, callback)` | Sends a raw Telegram API request.                                                                                               |
| `send_message(params)`      | Sends a message using the Telegram API. `params` is a dictionary containing message details.                                          |
| `send_text(peer_id, text, **kwargs)` | Sends a simple text message.                                                                                                    |
| `send_photo(peer_id, path, **kwargs)` | Sends a photo from a local file path.                                                                                         |
| `send_document(peer_id, path, **kwargs)` | Sends a generic file/document.                                                                                             |
| `send_video(peer_id, path, **kwargs)` | Sends a video file.                                                                                                       |
| `send_audio(peer_id, path, **kwargs)` | Sends an audio file.                                                                                                      |
| `edit_message(message_obj, **kwargs)` | Edits an existing message.                                                                                                 |
| `get_account_instance()` | Returns the current `AccountInstance`.                                                                                                 |
| `get_messages_controller()` | Returns the `MessagesController`.                                                                                                     |
| `get_contacts_controller()` | Returns the `ContactsController`.                                                                                                     |
| `get_media_data_controller()` | Returns the `MediaDataController`.                                                                                                   |
| `get_connections_manager()` | Returns the `ConnectionsManager`.                                                                                                     |
| `get_location_controller()` | Returns the `LocationController`.                                                                                                     |
| `get_notifications_controller()` | Returns the `NotificationsController`.                                                                                             |
| `get_messages_storage()`    | Returns the `MessagesStorage`.                                                                                                         |
| `get_send_messages_helper()` | Returns the `SendMessagesHelper`.                                                                                                       |
| `get_file_loader()`         | Returns the `FileLoader`.                                                                                                              |
| `get_secret_chat_helper()`  | Returns the `SecretChatHelper`.                                                                                                        |
| `get_download_controller()` | Returns the `DownloadController`.                                                                                                      |
| `get_notifications_settings()` | Returns the `NotificationsSettings`.                                                                                               |
| `get_notification_center()` | Returns the `NotificationCenter`.                                                                                                      |
| `get_media_controller()`    | Returns the `MediaController`.                                                                                                         |
| `get_user_config()`         | Returns the `UserConfig`.                                                                                                              |

### Android Utils Module

| Function/Class        | Description                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| `run_on_ui_thread(callable, delay)` | Executes a function on the main UI thread. `delay` is in milliseconds.                                  |
| `log(data)`           | Logs a message to the Android logcat.                                                                     |
| `R(callable)`         | A proxy for Java's `Runnable` interface.                                                                |
| `OnClickListener(callable)` | A proxy for Android's `View.OnClickListener`.                                                         |
| `OnLongClickListener(callable)` | A proxy for Android's `View.OnLongClickListener`.                                                     |

### File Utils Module

| Function           | Description                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| `get_plugins_dir()`  | Returns the path to the plugins directory.                                  |
| `get_cache_dir()`    | Returns the path to the main cache directory.                               |
| `get_files_dir()`    | Returns the path to the files directory.                                    |
| `get_images_dir()`   | Returns the path to the images directory.                                   |
| `get_videos_dir()`   | Returns the path to the videos directory.                                   |
| `get_audios_dir()`   | Returns the path to the audios directory.                                   |
| `get_documents_dir()`| Returns the path to the documents directory.                                |
| `ensure_dir_exists(path)` | Ensures that a directory exists.                                           |
| `list_dir(path, recursive, include_files, include_dirs, extensions)` | Lists the contents of a directory.                                   |
| `write_file(path, content)` | Writes a string to a file.                                                |
| `read_file(path)`    | Reads the content of a file.                                                |
| `delete_file(path)`  | Deletes a file.                                                             |

### Hook Utils Module

| Function                   | Description                                                 |
| -------------------------- | ----------------------------------------------------------- |
| `find_class(class_name)`   | Safely finds and returns a Java class object by its name.   |
| `get_private_field(obj, field_name)` | Retrieves the value of a private instance field.            |
| `set_private_field(obj, field_name, new_value)` | Modifies the value of a private instance field.           |
| `get_static_private_field(clazz, field_name)` | Retrieves the value of a static private field.            |
| `set_static_private_field(clazz, field_name, new_value)` | Modifies the value of a static private field.           |

### Markdown Utils Module

| Function/Class     | Description                                                                              |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `parse_markdown(markdown_text)` | Parses a Markdown string and returns a `ParsedMessage` object.                     |
| `ParsedMessage`    | A class with `text` and `entities` attributes.                                         |
| `RawEntity`        | A class representing a formatting entity, with a `to_tlrpc_object()` method.              |

### UI Components Reference

#### Alert Dialogs (`ui.alert`)
| Class/Function          | Description                                                              |
| ----------------------- | ------------------------------------------------------------------------ |
| `AlertDialogBuilder(context, progress_style, resources_provider)` | A wrapper for creating and managing Telegram-style alert dialogs. `progress_style` can be `ALERT_TYPE_MESSAGE`, `ALERT_TYPE_LOADING`, or `ALERT_TYPE_SPINNER`. |

#### Bulletin Notifications (`ui.bulletin`)
| Class/Function                | Description                                                          |
| ----------------------------- | -------------------------------------------------------------------- |
| `BulletinHelper.show_info(message, fragment)` | Shows a bulletin with a default info icon.                         |
| `BulletinHelper.show_error(message, fragment)` | Shows a bulletin with a default error/alert icon.                    |
| `BulletinHelper.show_success(message, fragment)` | Shows a bulletin with a default success/check icon.                  |
| `BulletinHelper.show_simple(text, icon_res_id, fragment)` | Shows a single-line bulletin with a custom icon.                |
| `BulletinHelper.show_two_line(title, subtitle, icon_res_id, fragment)` | Shows a two-line bulletin with a custom icon.                   |
| `BulletinHelper.show_with_button(text, icon_res_id, button_text, on_click, fragment, duration)` | Shows a bulletin with a button.                                    |
| `BulletinHelper.show_undo(text, on_undo, on_action, subtitle, fragment)` | Shows an "Undo"-style bulletin.                                      |
| `BulletinHelper.show_copied_to_clipboard(message, fragment)` | Shows a "Text copied to clipboard" bulletin.                      |

#### Settings Components (`ui.settings`)

| Class Name              | Description and Parameters                                                                                                                                                                                                                                                                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SettingsFragment(title, items, searchable, on_search, on_create, on_resume, on_destroy, resources_provider)` | The main container for a settings screen. **`title`**: The text displayed in the action bar. **`items`**: A list of settings component objects (e.g., `Header`, `Button`, `Checkbox`). **`searchable`**: Boolean, if true, adds a search icon. **`on_search`**: A function to handle the search query. **`on_create`**, **`on_resume`**, **`on_destroy`**: Fragment lifecycle callbacks. |
| `Header(text)`            | A simple header text to group settings. **`text`**: The header title.                                                                                                                                                                                                                                                                              |
| `Button(text, icon, on_click, on_long_click)` | A standard clickable text item. **`text`**: The main text of the button. **`icon`**: An optional resource ID for an icon on the left. **`on_click`**, **`on_long_click`**: Callback functions for click and long-click events.                                                                                                    |
| `Subtitle(text)`          | A small, gray-colored text typically used for providing descriptions or help text under other components. **`text`**: The descriptive text.                                                                                                                                                                                                           |
| `Checkbox(text, checked, on_change)` | A settings item with a checkbox. **`text`**: The label for the checkbox. **`checked`**: A boolean indicating the initial state. **`on_change`**: A callback function that is triggered when the state changes. The function receives the new boolean state as an argument.                                                                  |
| `RadioList(items, selected, on_change)` | A group of radio buttons, typically presented in an alert dialog when the setting is clicked. **`items`**: A list of strings representing the choices. **`selected`**: The index of the initially selected item. **`on_change`**: A callback function that is triggered with the index of the newly selected item.                     |
| `EditText(text, value, on_change)` | A setting that opens a dialog for text input. **`text`**: The label for the setting. **`value`**: The initial text value. **`on_change`**: A callback function that is triggered with the new text value when the user confirms the input.                                                                                                |
| `ColorPicker(text, color, on_change)` | A setting that opens a color picker dialog. **`text`**: The label for the setting. **`color`**: The initial color value (as an integer). **`on_change`**: A callback function that is triggered with the newly selected color integer.                                                                                                  |
| `Divider()`               | A visual separator line. It takes no parameters.                                                                                                                                                                                                                                                                                                  |

### Hook Reference: API Method Names

The `add_hook(hook_name, callback)` function allows plugins to intercept and modify outgoing Telegram API requests. Here are commonly used hooks:

| Hook Name                       | Trigger                                                                  | Common Use Cases                                                                                                          |
| ------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `TL_messages_sendMessage`         | When a message (text, media, etc.) is being sent.                        | Modifying outgoing messages, creating auto-responders, logging sent messages, implementing custom commands.             |
| `TL_messages_editMessage`         | When a message is being edited.                                          | Tracking message edits, preventing certain edits, adding custom syntax.                                                   |
| `TL_messages_sendMedia`           | When media (photo, video, document) is being sent.                       | Compressing media, adding watermarks, analyzing files before sending.                                                     |
| `TL_messages_setTyping`           | When your client sends a "typing..." status to a chat.                   | Creating custom typing statuses (e.g., "recording audio...", "thinking..."), or hiding your typing status.                 |
| `TL_account_updateStatus`         | When your online/offline status is updated.                              | Freezing your "last seen" status, or setting a custom status that persists.                                               |
| `TL_messages_getDialogs`          | When the client requests the list of chats/dialogs.                      | Filtering the chat list, adding custom folders or categories (client-side only), sorting dialogs in a custom way.      |
| `TL_messages_getHistory`          | When the message history for a chat is loaded.                           | Filtering incoming messages, highlighting specific content, injecting custom client-side messages into the chat view. |
| `TL_users_getUsers`               | When information about one or more users is requested.                   | Modifying user profiles as they are displayed, adding custom badges or titles (client-side).                             |
| `TL_channels_getFullChannel`      | When detailed information about a channel or supergroup is requested.    | Displaying additional administrative information, adding custom buttons or info panels to the channel view.               |
| `TL_messages_readHistory`         | When a chat's history is marked as read.                                 | Preventing read receipts from being sent, creating "read-later" functionality.                                            |
| `TL_messages_deleteMessages`      | When one or more messages are deleted.                                   | Preventing message deletion (client-side), logging deleted messages.                                                      |

### Advanced Patterns & Best Practices

#### 1. Xposed Method Hooking

For ultimate control, plugins can hook directly into Java/Kotlin methods using the `XposedHook` class:

```python
XposedHook(
    class_name,
    method_name,
    callback,
    return_value=None,
    before=True
)
```

- `class_name`: The fully qualified name of the Java class (e.g., `org.telegram.ui.LaunchActivity`)
- `method_name`: The name of the method to hook
- `callback`: A Python function that receives `param` argument with:
  - `param.args`: List of method arguments (can modify in-place)
  - `param.method`: The original Java `Method` object
  - `param.thisObject`: The instance of the class (`this`)
  - `param.return_value`: The value that will be returned
  - `param.set_result(value)`: Function to set new return value and prevent original method

Example - Disabling camera shutter sound:
```python
from Exteragram.hooks import XposedHook

SHUTTER_CLICK = 0

def silent_shutter_callback(param):
    sound_id = param.args[0]
    if sound_id == SHUTTER_CLICK:
        param.set_result(None)

def on_load():
    XposedHook(
        "android.media.MediaActionSound",
        "play",
        silent_shutter_callback
    )
```

#### 2. Client-Side Caching

Using `zwyLib` for shared caching:
```python
# Save data
zwyLib.cache.set("my_plugin_key", {"user_id": 123, "note": "Hello"})

# Read data
data = zwyLib.cache.get("my_plugin_key", default_value={})

# Delete data
zwyLib.cache.delete("my_plugin_key")
```

#### 3. Dynamic UI Manipulation

- Use `get_private_field` from `hook_utils` to access private UI components
- Use `run_on_ui_thread` from `android_utils` for safe UI updates
- Add new `View` objects to existing layouts

#### 4. Handling Intents (Sharing)

```python
from java.io import File
from android.content import Intent
from android.support.v4.content import FileProvider

# Share a text file
context = get_application_context()
file = File(file_path)
uri = FileProvider.getUriForFile(context, "org.telegram.messenger.provider", file)

intent = Intent(Intent.ACTION_SEND)
intent.setType("text/plain")
intent.putExtra(Intent.EXTRA_STREAM, uri)
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)

chooser = Intent.createChooser(intent, "Share via...")
chooser.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
context.startActivity(chooser)
```

### Xposed Method Hooking (Advanced)

#### Hook Handler Base Classes

For clarity and correctness, create handlers by inheriting from these base classes:

- `MethodHook`: Run code before and/or after the original method executes
- `MethodReplacement`: Completely replace the original method's logic

#### Filters

You can set filters to control whether hook callbacks execute using `@hook_filters` decorator:

```python
from base_plugin import MethodHook, hook_filters, HookFilter

class ExampleFilter(MethodHook):
    # Run only if first argument is null
    @hook_filters(HookFilter.ArgumentIsNull(0))
    def before_hooked_method(self, param):
        ...

    # Run only if result is null
    @hook_filters(HookFilter.RESULT_IS_NULL)
    def after_hooked_method(self, param):
        ...

class ExampleComplexFilter(MethodHook):
    # Run if first argument is "TEST" OR second argument is true
    @hook_filters(HookFilter.Or(
        HookFilter.ArgumentEqual(0, "TEST"), 
        HookFilter.ArgumentIsTrue(1)
    ))
    def before_hooked_method(self, param):
        ...
```

#### Filter Types

- `RESULT_IS_NULL`, `RESULT_IS_TRUE`, `RESULT_IS_FALSE`, `RESULT_NOT_NULL`
- `ResultIsInstanceOf(clazz)`, `ResultEqual(value)`, `ResultNotEqual(value)`
- `ArgumentIsNull(index)`, `ArgumentNotNull(index)`, `ArgumentIsFalse(index)`, `ArgumentIsTrue(index)`
- `ArgumentEqual(index, value)`, `ArgumentNotEqual(index, value)`, `ArgumentIsInstanceOf(index, clazz)`
- `And(*filters)`, `Or(*filters)`, `Not(filter)`

#### Complete Hooking Process

```python
from base_plugin import MethodHook, MethodReplacement
from hook_utils import find_class
from java import jint

# 1. Find Target Method
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if not ActionBarClass:
    return

CharSequenceClass = find_class("java.lang.CharSequence")
method_to_hook = ActionBarClass.getClass().getDeclaredMethod("setTitle", CharSequenceClass)
method_to_hook.setAccessible(True)

# 2. Implement Hook Handler
class TitleLoggerHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        title = param.args[0]
        self.plugin.log(f"Title being set to: {title}")
        param.args[0] = f"[Hooked] {title}"

    def after_hooked_method(self, param):
        self.plugin.log("Title has been set.")

# 3. Apply Hook
handler_instance = TitleLoggerHook(self)
self.unhook_obj = self.hook_method(method_to_hook, handler_instance, priority=10)

# 4. Hook Multiple Methods/Constructors
# Hook all methods with specific name
unhook_list = self.hook_all_methods(MyViewClass, "onMeasure", on_measure_handler)

# Hook all constructors
unhook_list = self.hook_all_constructors(MyClass, constructor_handler)

# 5. Manual Unhooking
if self.unhook_obj:
    self.unhook_method(self.unhook_obj)
```

#### Practical Examples

**Example 1: Modifying Arguments**
```python
from base_plugin import MethodHook
from hook_utils import find_class
from java import jint

class ToastHook(MethodHook):
    def before_hooked_method(self, param):
        # Method: makeText(Context context, CharSequence text, int duration)
        original_text = param.args[1]
        param.args[1] = f"(Plugin) {original_text}"

# Apply hook
ToastClass = find_class("android.widget.Toast")
ContextClass = find_class("android.content.Context")
CharSequenceClass = find_class("java.lang.CharSequence")
make_text_method = ToastClass.getClass().getDeclaredMethod("makeText", ContextClass, CharSequenceClass, jint)
self.hook_method(make_text_method, ToastHook())
```

**Example 2: Changing Return Value**
```python
class BuildVarsHook(MethodHook):
    def after_hooked_method(self, param):
        original_result = param.getResult()
        param.setResult(False)  # Always return False

# Apply to BuildVars.isMainApp()
BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
is_main_app_method = BuildVarsClass.getClass().getDeclaredMethod("isMainApp")
self.hook_method(is_main_app_method, BuildVarsHook())
```

**Example 3: Method Replacement**
```python
from base_plugin import MethodReplacement

class NoOpLogger(MethodReplacement):
    def replace_hooked_method(self, param):
        # Do nothing - original method never called
        return None

# Apply to disable logging
FileLogClass = find_class("org.telegram.messenger.FileLog")
log_method = FileLogClass.getClass().getDeclaredMethod("d", JString)
self.hook_method(log_method, NoOpLogger())
```

---

## Advanced Android UI Utilities

### Java Interface Wrappers

#### R (Runnable Proxy)
```python
from android_utils import R, log, run_on_ui_thread

def my_task():
    print("This task will run.")

# Create Runnable instance
runnable_instance = R(my_task)
```

#### OnClickListener
```python
from android_utils import OnClickListener, log
from android.view import View

def handle_button_click(view: View):
    log(f"Button {view.getId()} was clicked!")

button.setOnClickListener(OnClickListener(handle_button_click))
```

#### OnLongClickListener
```python
from android_utils import OnLongClickListener, log

def handle_button_long_click(view: View):
    log(f"Button {view.getId()} was long-clicked!")
    return True

button.setOnLongClickListener(OnLongClickListener(handle_button_long_click))
```

### Utility Functions

#### run_on_ui_thread
```python
from android_utils import run_on_ui_thread

def update_ui_content():
    text_view.setText("Updated from Python on UI thread")

# Run immediately
run_on_ui_thread(update_ui_content)

# Run with delay (500ms)
run_on_ui_thread(update_ui_content, 500)
```

#### log
```python
from android_utils import log

# Log simple messages
log("This is a simple log message.")
log(f"User count: {123}")
log(True)

# Log objects (detailed structure)
log(user_object)  # Detailed user object info
log(some_list)    # List contents details
```

---

## Enhanced Client Utilities

### Background Queues

Available queues for `run_on_queue`:

- `STAGE_QUEUE` = "stageQueue"                # Critical, sequential operations
- `GLOBAL_QUEUE` = "globalQueue"              # General purpose background tasks
- `CACHE_CLEAR_QUEUE` = "cacheClearQueue"    # Cache management tasks
- `SEARCH_QUEUE` = "searchQueue"              # Search operations
- `PHONE_BOOK_QUEUE` = "phoneBookQueue"      # Phone book and contact sync
- `THEME_QUEUE` = "themeQueue"                # Theme application and processing
- `EXTERNAL_NETWORK_QUEUE` = "externalNetworkQueue" # External network requests
- `PLUGINS_QUEUE` = "pluginsQueue"            # **Default queue** for plugin tasks

```python
from client_utils import run_on_queue, GLOBAL_QUEUE, PLUGINS_QUEUE

# Run on default queue
run_on_queue(lambda: my_long_task("data"))

# Run on specific queue with delay
run_on_queue(lambda: my_long_task("other_data"), GLOBAL_QUEUE, 2500)
```

### Controller Access

```python
from client_utils import (
    get_account_instance, get_messages_controller, get_contacts_controller,
    get_media_data_controller, get_connections_manager, get_location_controller,
    get_notifications_controller, get_messages_storage, get_send_messages_helper,
    get_file_loader, get_secret_chat_helper, get_download_controller,
    get_notifications_settings, get_notification_center, get_media_controller,
    get_user_config
)

# Get instances
account_instance = get_account_instance()
messages_controller = get_messages_controller()
connections_manager = get_connections_manager()
send_helper = get_send_messages_helper()
user_cfg = get_user_config()

# Example usage
if user_cfg.getCurrentUser():
    user_name = user_cfg.getCurrentUser().first_name

messages_controller.loadDialogs(0, 50, True)
```

---

## Enhanced File Operations

### Standard Directories

```python
from file_utils import (
    get_plugins_dir, get_cache_dir, get_files_dir, get_images_dir,
    get_videos_dir, get_audios_dir, get_documents_dir
)

# Get standard paths
plugins_path = get_plugins_dir()
cache_path = get_cache_dir()
files_path = get_files_dir()
images_path = get_images_dir()
videos_path = get_videos_dir()
audios_path = get_audios_dir()
documents_path = get_documents_dir()
```

### Directory Operations

```python
from file_utils import ensure_dir_exists, list_dir, get_plugins_dir
import os

# Ensure directory exists
my_plugin_data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(my_plugin_data_dir)

# List directory contents
image_files = list_dir(
    path=get_images_dir(),
    extensions=[".jpg", ".png"]
)

# Recursive listing
cache_subdirs = list_dir(
    path=get_cache_dir(),
    recursive=True,
    include_files=False,
    include_dirs=True
)
```

### File Operations

```python
from file_utils import write_file, read_file, delete_file, get_plugins_dir
import os

# Write file
data_to_save = "Hello, World!"
my_data_path = os.path.join(get_plugins_dir(), "my_plugin_data", "data.log")
write_file(my_data_path, data_to_save)

# Read file
config_content = read_file(my_data_path)

# Delete file
was_deleted = delete_file("/path/to/temp_file.tmp")
```

---

## Enhanced Alert Dialog Builder

### Dialog Types

```python
from ui.alert import AlertDialogBuilder

# Standard message dialog (default)
builder = AlertDialogBuilder(activity)

# Loading dialog with spinner
loading_builder = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_SPINNER)
loading_builder.set_title("Loading Data...")
loading_builder.set_message("Please wait...")
loading_builder.set_cancelable(False)

# Progress dialog
progress_builder = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_LOADING)
progress_builder.set_title("Downloading...")
progress_builder.set_progress(50)  # 0-100
```

### Advanced Dialog Features

```python
# Custom view
from android.view import View
custom_view = ... # Create Android View
builder.set_view(custom_view)

# List items with icons
items = ["Option A", "Option B", "Option C"]
icons = [icon_a, icon_b, icon_c]  # Resource IDs
builder.set_items(items, on_item_click, icons)

# Custom buttons
builder.set_positive_button("OK", on_positive_click)
builder.set_negative_button("Cancel", on_negative_click)
builder.set_neutral_button("Later", on_neutral_click)

# Button styling
builder.make_button_red(AlertDialogBuilder.BUTTON_NEGATIVE)

# Listeners
builder.set_on_dismiss_listener(on_dismiss)
builder.set_on_cancel_listener(on_cancel)
builder.set_on_back_button_listener(on_back)

# Appearance
builder.set_dim_enabled(False)
builder.set_blurred_background(True)
builder.set_canceled_on_touch_outside(False)
```

---

## Enhanced Bulletin Helper

### Standard Bulletin Types

```python
from ui.bulletin import BulletinHelper

# Standard bulletins
BulletinHelper.show_info("Information message")
BulletinHelper.show_error("An error occurred")
BulletinHelper.show_success("Action completed successfully!")
```

### Custom Bulletins

```python
# Simple custom bulletin
BulletinHelper.show_simple("Processing...", R_tg.raw.timer)

# Two-line bulletin
BulletinHelper.show_two_line(
    "Download Complete", 
    "File saved to gallery.", 
    R_tg.raw.ic_download_done
)

# Bulletin with button
def open_settings():
    print("Settings opened!")

BulletinHelper.show_with_button(
    "Plugin settings updated.",
    R_tg.raw.info,
    "Configure",
    open_settings,
    duration=BulletinHelper.DURATION_PROLONG
)

# Undo-style bulletin
def perform_action():
    print("Action committed")

def undo_action():
    print("Action undone")

BulletinHelper.show_undo(
    "Item moved to trash",
    on_undo=undo_action,
    on_action=perform_action
)
```

### Contextual Bulletins

```python
# Clipboard
BulletinHelper.show_copied_to_clipboard("Text copied to clipboard")

# Link copied
BulletinHelper.show_link_copied()

# File saved
BulletinHelper.show_file_saved_to_gallery(is_video=True, amount=3)
BulletinHelper.show_file_saved_to_downloads("MUSIC", amount=2)

# Duration constants
DURATION_SHORT = 1500    # 1.5 seconds
DURATION_LONG = 2750     # 2.75 seconds  
DURATION_PROLONG = 5000  # 5 seconds
```

---

## Common Telegram Classes Reference

### Key Classes for Plugin Development

#### Core UI Classes
- **LaunchActivity**: App initialization and custom link handling
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java`
  - Use for: App startup hooks, link interception

- **ProfileActivity**: User and channel profile screens  
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java`
  - Use for: Profile menu items, user information access

- **ChatActivity**: Chat rendering and functionality
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`
  - Use for: Message hooks, chat-specific functionality

#### Message Handling
- **MessageObject**: Wrapper for `TLRPC.Message`
  - Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java`
  - Use for: Message analysis and manipulation

- **ChatMessageCell**: Message rendering component
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java`
  - Use for: UI manipulation, message display changes

#### Utility Classes
- **AndroidUtilities**: General utility functions
  - Path: `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java`
  - Use for: File operations, UI utilities, formatting

- **MessagesController**: App state and Telegram request management
  - Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java`
  - Use for: Message operations, dialog management

- **MessagesStorage**: Local database state management
  - Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java`
  - Use for: Database operations, local data access

#### Network and Communication
- **SendMessagesHelper**: Message sending operations
  - Path: `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`
  - Use for: Sending messages, file uploads

- **TLRPC**: All Telegram request models
  - Path: `TMessagesProj/src/main/java/org/telegram/tgnet/`
  - Use for: API request/response handling

#### UI Components
- **BulletinFactory**: Bottom notification messages
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java`
  - Use for: Custom notifications

- **AlertDialog**: Dialog overlay support
  - Path: `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java`
  - Use for: Custom dialogs with action buttons

---

## Complete Plugin Development Tutorial

### Building a Complete Weather Plugin

Here's the step-by-step creation of a production-ready weather plugin:

```python
import requests
from android_utils import log
from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import run_on_queue, run_on_ui_thread, get_last_fragment, send_message
from ui.alert import AlertDialogBuilder
from typing import Any, Optional

__id__ = "weather_advanced"
__name__ = "Weather Advanced"
__description__ = "Provides current weather information asynchronously [.wt]"
__author__ = "Your Name"
__version__ = "2.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"

API_BASE_URL = "https://wttr.in"
API_HEADERS = {"User-Agent": "Mozilla/5.0", "Accept": "application/json"}

class WeatherAdvancedPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.progress_dialog_builder: Optional[AlertDialogBuilder] = None

    def on_plugin_load(self):
        self.add_on_send_message_hook()
        self.log("Weather plugin loaded successfully!")

    def on_plugin_unload(self):
        if self.progress_dialog_builder:
            try:
                self.progress_dialog_builder.dismiss()
            except:
                pass
        self.log("Weather plugin unloaded")

    def create_settings(self):
        from ui.settings import Header, Switch, Input, Divider
        return [
            Header(text="Weather Settings"),
            Switch(
                key="use_celsius",
                text="Use Celsius",
                default=True,
                subtext="Display temperature in Celsius"
            ),
            Input(
                key="default_city",
                text="Default City",
                default="Moscow",
                subtext="City to use when none specified"
            ),
            Divider(),
            Header(text="Advanced"),
            Switch(
                key="show_detailed_forecast",
                text="Show Detailed Forecast",
                default=False,
                subtext="Include humidity and wind information"
            )
        ]

    def fetch_weather_data(self, city: str):
        """Fetch weather data from API with error handling"""
        try:
            url = f"{API_BASE_URL}/{city}?format=j1"
            response = requests.get(url, headers=API_HEADERS, timeout=10)
            if response.status_code != 200:
                log(f"Failed to fetch weather data for '{city}' (status code: {response.status_code})")
                return None
            return response.json()
        except Exception as e:
            log(f"Weather API error: {str(e)}")
            return None

    def format_weather_data(self, data: dict, query_city: str):
        """Format weather data according to settings"""
        try:
            area_info = data.get("nearest_area", [{}])[0]
            city = area_info.get("areaName", [{}])[0].get("value", query_city)
            region = area_info.get("region", [{}])[0].get("value", "")
            country = area_info.get("country", [{}])[0].get("value", "")
            
            location_parts = [city]
            if region:
                location_parts.append(region)
            if country:
                location_parts.append(country)
            location_str = ", ".join(location_parts)
            
            result_parts = [f"Weather in {location_str}:\n\n"]
            current = data.get("current_condition", [{}])[0]
            
            # Temperature handling
            temp_c = current.get("temp_C", "N/A")
            temp_f = current.get("temp_F", "N/A")
            feels_like_c = current.get("FeelsLikeC", "N/A")
            feels_like_f = current.get("FeelsLikeF", "N/A")
            
            if self.get_setting("use_celsius", True):
                result_parts.append(f"• Temperature: {temp_c}°С (Feels like: {feels_like_c}°С)\n")
            else:
                result_parts.append(f"• Temperature: {temp_f}°F (Feels like: {feels_like_f}°F)\n")
            
            condition = current.get("weatherDesc", [{}])[0].get("value", "Unknown")
            result_parts.append(f"• Condition: {condition}\n")
            
            # Optional detailed information
            if self.get_setting("show_detailed_forecast", False):
                humidity = current.get("humidity", "N/A")
                wind_speed = current.get("windspeedKmph", "N/A")
                wind_dir = current.get("winddir16Point", "N/A")
                pressure = current.get("pressure", "N/A")
                visibility = current.get("visibility", "N/A")
                
                result_parts.extend([
                    f"• Humidity: {humidity}%\n",
                    f"• Wind: {wind_speed} km/h ({wind_dir})\n",
                    f"• Pressure: {pressure} mb\n",
                    f"• Visibility: {visibility} km\n"
                ])
            
            local_time = current.get("localObsDateTime", "N/A")
            result_parts.append(f"\nUpdated: {local_time} (local time)")
            
            return "".join(result_parts)
        except Exception as e:
            log(f"Error formatting weather data: {str(e)}")
            return f"Error processing weather data: {str(e)}"

    def _process_weather_request(self, city: str, peer_id: Any):
        """Process weather request in background thread"""
        data = self.fetch_weather_data(city)
        
        if not data:
            message_content = f"Failed to fetch weather data for '{city}'. Please check the city name and try again."
        else:
            message_content = self.format_weather_data(data, city)
        
        message_params = {
            "message": message_content,
            "peer": peer_id
        }
        
        def _send_message_and_dismiss_dialog():
            if self.progress_dialog_builder:
                try:
                    self.progress_dialog_builder.dismiss()
                    self.progress_dialog_builder = None
                except:
                    pass
            send_message(message_params)
        
        run_on_ui_thread(_send_message_and_dismiss_dialog)

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        """Handle outgoing message hook for weather commands"""
        if not isinstance(params.message, str) or not params.message.startswith(".wt"):
            return HookResult()
        
        try:
            # Parse command
            parts = params.message.strip().split(" ", 1)
            city = parts[1].strip() if len(parts) > 1 else self.get_setting("default_city", "Moscow")
            
            if not city:
                params.message = "Usage: .wt [city_name] or set default city in settings"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            # Validate city name (basic)
            if len(city) > 50 or not city.replace(" ", "").replace("-", "").replace(".", "").isalnum():
                params.message = "Invalid city name. Please use only letters, numbers, spaces, hyphens, and dots."
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            # Show loading dialog
            current_fragment = get_last_fragment()
            if not current_fragment:
                log("WeatherPlugin: Could not get current fragment to show dialog.")
                # Fallback to simple message modification instead of canceling
                params.message = f"Fetching weather for {city}..."
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            current_activity = current_fragment.getParentActivity()
            if not current_activity:
                log("WeatherPlugin: Could not get current activity to show dialog.")
                params.message = f"Fetching weather for {city}..."
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            self.progress_dialog_builder = AlertDialogBuilder(
                current_activity,
                AlertDialogBuilder.ALERT_TYPE_SPINNER
            )
            self.progress_dialog_builder.set_title("Fetching Weather...")
            self.progress_dialog_builder.set_message(f"Getting current weather for {city}")
            self.progress_dialog_builder.set_cancelable(False)
            self.progress_dialog_builder.show()
            
            # Process in background
            run_on_queue(lambda: self._process_weather_request(city, params.peer))
            
            # Cancel original message since we'll send the result programmatically
            return HookResult(strategy=HookStrategy.CANCEL)
        
        except Exception as e:
            log(f"WeatherPlugin: Error in on_send_message_hook: {str(e)}")
            params.message = f"Weather error: {str(e)}. Please try again later."
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

This complete example demonstrates:
- Proper error handling and user feedback
- Settings integration for user customization
- Background thread processing for API calls
- UI thread updates for dialog management
- Input validation and sanitization
- Progressive enhancement with fallback mechanisms
- Comprehensive logging for debugging
- Clean resource management and cleanup

---

---

## Conclusion

This guide provides a comprehensive foundation for developing Exteragram plugins. Key principles to remember:

1. **Always handle errors gracefully** - Never let your plugin crash the app
2. **Use background threads** for long operations - Keep the UI responsive
3. **Cache expensive operations** - Improve performance and reduce API calls
4. **Follow the plugin lifecycle** - Proper initialization and cleanup
5. **Provide good settings UI** - Make your plugin configurable
6. **Test thoroughly** - Especially with real data and edge cases
7. **Log appropriately** - Help users and yourself debug issues

For more examples and the latest updates, check the [official Exteragram plugins repository](https://t.me/exteraPlugins) and community channels.

---

**Happy Plugin Development! 🎉**
