
# Exteragram Plugin Collection

This repository contains multiple plugins for [Exteragram](https://t.me/exteraGramCI), a Telegram client. Each plugin adds unique features to enhance your chat experience. Below you'll find a description of each plugin and instructions for installation and usage.

---

## Plugins

### 1. Secure Chat Plugin (`secure_chat.plugin`)

**Description:**
Provides end-to-end encrypted chat functionality for Exteragram. Messages sent with `.enc` or `.encrypt` commands are encrypted using ChaCha20Poly1305 and automatically decrypted on receipt. Per-chat keys are managed securely, and the plugin integrates with the UI for notifications and settings.

**Features:**
- End-to-end encryption for messages
- Per-chat key management
- Automatic decryption of incoming encrypted messages
- UI integration for notifications and settings

**Commands:**
- `.enc <message>` or `.encrypt <message>`: Encrypt and send a message
- `.setkey <key>`: Set encryption key for current chat
- `.exportkey`: Show current chat's key
- `.importkey <key>`: Import key for current chat
- `.showkeys`: List all set keys (truncated)

**Requirements:**
- Exteragram v11.12.0 or higher

---

### 2. Setsu Plugin Lib (`setsu_plugin_lib.plugin`)

**Description:**
Allows you to fetch and install plugins directly from a GitHub repository. The plugin provides a settings UI to configure the repository owner, name, and branch, and displays available plugins for installation. Useful for managing and updating plugins from a central source.

**Features:**
- Fetch list of plugins from GitHub
- Install plugins with one click
- Configurable repository owner, name, and branch
- UI integration for plugin management

**Requirements:**
- Exteragram v11.12.0 or higher

---

## Installation

1. Download the desired `.plugin` file(s) from this repository or releases.
2. Open Exteragram and go to the plugins section.
3. Install the plugin by selecting the downloaded file.
4. Enable the plugin in the plugin menu.

## Usage

Refer to each plugin's section above for specific commands and features. Settings for each plugin can be accessed via Exteragram's plugin settings menu.

## Contributing

Contributions are welcome! Fork the repository and submit a pull request with your improvements or new plugins.

## License

This project is licensed under the MIT License. See the LICENSE file for details.

## Support

For issues or questions, please open an issue on the GitHub repository.
