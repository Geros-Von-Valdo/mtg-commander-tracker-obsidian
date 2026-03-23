# MTG Commander Tracker for Obsidian

A lightweight,tracker built directly inside [Obsidian](https://obsidian.md/) to manage your next Magic: The Gathering Commander playthrough.

**Why did I build this?**
As someone who loves experimenting with a ton of different decks in a short amount of time on Tabletop Simulator, I needed a way to keep track of what I was brewing, playing, and winning with. I created this little project to stay organized and keep all my deck stats in one place.

Powered by **DataviewJS**, this tracker automatically fetches real-time data from the Scryfall API and EDHREC, providing a clean, interactive, and locally persistent UI right in your vault.

##  Features

- **Automated Data Scraping:** Simply paste an EDHREC URL, and the script fetches the commander's name, card art, color identity, rank, and top themes automatically.
- **Local Persistence:** All your deck data is saved in a local `commanders.json` file. No databases to configure, everything stays in your vault.
- **Interactive Dashboard:** - Sort by rank, name, colors, or your win rate.
  - Track "Played" status and "Wins" directly from the table.
- **Click-to-Zoom Lightbox:** Click on any commander's art to open a high-resolution, full-screen preview.
- **Theme Agnostic:** Designed with Obsidian's native CSS variables to look great on any Light or Dark theme without breaking the layout.

---

##  Getting Started

Setting up the tracker is quick and requires only one community plugin.

### Prerequisites

1. Install the **[Dataview](https://github.com/blacksmithgu/obsidian-dataview)** plugin from the Obsidian Community Plugins store.
2. Go to `Settings > Dataview` and toggle **Enable JavaScript Queries** to `ON`.

### Installation

1. **Clone or Download** this repository into your Obsidian Vault.
2. Ensure you have the following structure in the same folder:
   - `Commander Tracker.md` (The main dashboard file containing the DataviewJS script).
   - `commanders.json` (The local database file. Keep it as an empty array `[]` for a fresh start).
3. Open `Commander Tracker.md` in **Reading View** or **Live Preview**. The dashboard will initialize automatically.

---

##  Usage

1. Go to [EDHREC](https://edhrec.com/) and find a commander you want to track.
2. Copy the URL (e.g., `https://edhrec.com/commanders/nekusar-the-mindrazer`).
3. Paste the link into the input field at the top of your Obsidian dashboard.
4. Click **Add**. 
5. The tracker will fetch the metadata, download the Scryfall image reference, and update your list instantly.

### Under the Hood
This tool relies on client-side fetching. Images are loaded dynamically via Scryfall's CDN and cached by Obsidian, ensuring optimal performance without bloating your vault size. The layout includes a custom CSS injection to bypass default Obsidian reading margins, maximizing your screen real estate.

---
### ⚖️ Legal Disclaimer

*This project is unofficial Fan Content permitted under the Fan Content Policy. Not approved/endorsed by Wizards. Portions of the materials used are property of Wizards of the Coast. ©Wizards of the Coast LLC. Magic: The Gathering and its art are property of Wizards of the Coast. No copyright infringement intended.*
