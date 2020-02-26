+++
title = "Nyanko"
description = "A cross platform anime client written in Rust and QML."
weight = 0

[extra]
git = "https://github.com/HiruNya/nyanko"
+++

Nyanko is an anime client primarily written in Rust.
The GUI is made in Qt Quick using QML via the [qmetaobject](https://github.com/woboq/qmetaobject-rs) crate.

The project is currently in its initial steps so right now the only features are:
- Searching [AniList] for anime.

Planned features:
- Allow adding, updating, and deleting entries from an [AniList] list.
- Support for multiple accounts.
- Allow locally stored lists (using SQLite).
- Watches the title of active windows to see whether an anime is being watched and if so, update the list.
- Cache search results that will show up as the user is typing (using SQLite).

[AniList]: https://anilist.co
