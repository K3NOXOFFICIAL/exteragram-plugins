# Secure Chat Plugin for Exteragram

## Overview

Secure Chat is an encryption plugin designed for Exteragram, a Telegram client. This plugin enhances the security of your conversations by providing end-to-end encryption features, ensuring that your messages remain private and protected from unauthorized access.

## Features

- **End-to-End Encryption**: Encrypts messages before sending and decrypts them upon receipt.
- **Secure Key Management**: Handles encryption keys securely within the plugin.
- **Compatibility**: Seamlessly integrates with Exteragram's interface.
- **Lightweight**: Minimal impact on performance.

## Installation

1. Download the plugin file `secure_chat.plugin` from the releases section.
2. Open Exteragram and navigate to the plugins section.
3. Install the plugin by selecting the downloaded file.
4. Enable the plugin in the plugin in the plugin menu.

## Usage

Once installed, the plugin will automatically encrypt messages in supported chats. No additional configuration is required for basic usage.

- To enable encryption for a chat: Right-click on the chat and select "Enable Secure Chat".
- To disable: Right-click and select "Disable Secure Chat".

## Commands

The plugin supports the following commands, which must be prefixed with a dot (.):

- `.setkey <key>`: Sets the encryption key for the current chat. This key is required to encrypt and decrypt messages in this chat. Replace `<key>` with your chosen password or passphrase.
- `.enc <message>` or `.encrypt <message>`: Encrypts the specified message using the set key and sends it. If no key is set for the chat, the command will be canceled with an error message.

Encrypted messages are automatically decrypted when received. If a message cannot be decrypted (e.g., no key set), it will display as "[NO KEY]" or "[ERROR]" followed by the encrypted text.

## Requirements

- Exteragram version 11.12.0 or higher (check compatibility in releases).


## Contributing

Contributions are welcome! Please fork the repository and submit a pull request with your improvements.

## License

This project is licensed under the MIT License. See the LICENSE file for details.

## Support

For issues or questions, please open an issue on the GitHub repository.