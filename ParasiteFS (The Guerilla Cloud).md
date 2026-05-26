🦠 ParasiteFS (The Guerilla Cloud)
Infinite, free cloud storage hidden inside Big Tech’s video servers
The Concept: ParasiteFS is a FUSE-based file system that gives you infinite, decentralized cloud storage by smuggling your encrypted data inside fake media uploads. You save a file to your local ~/Parasite folder. The engine encrypts it, encodes the binary data into the pixels of a procedurally generated static noise video, and automatically uploads it to YouTube, TikTok, or Twitter. It saves the URL to a local SQLite index. When you read the file, it streams the YouTube video in the background, decodes the pixel static back into binary, decrypts it, and serves the file.

Why it’s next level: It treats Big Tech platforms as an unwitting, free, encrypted RAID array. It’s the ultimate anti-capitalist data hack—using the unlimited storage budgets of surveillance giants to host your private backups. Tech Stack: Rust (FUSE), FFmpeg (binary-to-video steganography), libsodium (encryption), Playwright (headless uploads).

