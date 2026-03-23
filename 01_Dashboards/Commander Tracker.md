```dataviewjs
/**
 * MTG Commander Tracker - Obsidian Dashboard
 * Final Release with Status Feedback
 */

// --- 0. INITIALIZATION & SAFETY GUARDS ---
const currentFile = dv.current()?.file;
if (!currentFile) return; 

const currentFolder = currentFile.folder;
const jsonPath = currentFolder + "/commanders.json";
let data = [];
let sortCol = "rank"; 
let sortAsc = true;

// --- 1. CSS INJECTION ---
const styleHeader = document.createElement('style');
styleHeader.innerHTML = `
    .markdown-rendered.node-insert-event { max-width: 100% !important; padding: 0 !important; }
    .dataview.js-view { width: 100% !important; max-width: 100% !important; }
    .commander-table { width: 100%; border-collapse: collapse; margin-top: 10px; table-layout: auto; }
    .commander-table th { padding: 12px 8px; border-bottom: 2px solid var(--background-modifier-border-strong); cursor: pointer; font-size: 0.9em; }
    .commander-table td { padding: 8px; border-bottom: 1px solid var(--background-modifier-border); vertical-align: middle !important; text-align: center; }
    .commander-table img { cursor: zoom-in; transition: opacity 0.2s; display: block; margin: 0 auto; border-radius: 4px; }
    .commander-table img:hover { opacity: 0.7; }
    @keyframes zoomIn { from { opacity: 0; transform: scale(0.9); } to { opacity: 1; transform: scale(1); } }
`;
document.head.appendChild(styleHeader);

// --- 2. CORE FUNCTIONS ---

function getColorName(colorArray) {
    if (!colorArray || colorArray.length === 0) return "Colorless";
    const order = { 'W': 1, 'U': 2, 'B': 3, 'R': 4, 'G': 5 };
    let sorted = [...colorArray].sort((a, b) => (order[a] || 9) - (order[b] || 9));
    const combos = {
        "W,U": "Azorius", "U,B": "Dimir", "B,R": "Rakdos", "R,G": "Gruul", "W,G": "Selesnya",
        "W,B": "Orzhov", "U,R": "Izzet", "B,G": "Golgari", "W,R": "Boros", "U,G": "Simic",
        "W,U,B": "Esper", "U,B,R": "Grixis", "B,R,G": "Jund", "W,R,G": "Naya", "W,U,G": "Bant",
        "W,B,G": "Abzan", "W,U,R": "Jeskai", "U,B,G": "Sultai", "W,B,R": "Mardu", "U,R,G": "Temur",
        "W,U,B,R,G": "5 Colors"
    };
    const key = sorted.join(",");
    const m = { 'W': 'White', 'U': 'Blue', 'B': 'Black', 'R': 'Red', 'G': 'Green' };
    return combos[key] || (sorted.length > 1 ? sorted.join("/") : "Mono " + (m[sorted[0]] || sorted[0]));
}

async function fetchCommanderData(url) {
    const match = url.match(/\/(?:commanders|cards)\/([^/]+)/);
    if (!match) return null;
    const slug = match[1];
    const clean_url = `https://edhrec.com/commanders/${slug}`;
    const name_query = slug.replace(/-/g, ' ');

    try {
        const encodedName = encodeURIComponent(name_query);
        let sfResp = await requestUrl({ 
            url: `https://api.scryfall.com/cards/named?fuzzy=${encodedName}`,
            headers: { "User-Agent": "ObsidianMagicDashboard/1.0", "Accept": "application/json" }
        });
        let sfData = sfResp.json;
        let image_url = sfData.image_uris ? sfData.image_uris.normal : (sfData.card_faces ? sfData.card_faces[0].image_uris.normal : "");

        let edhResp = await requestUrl({ url: clean_url, headers: { "User-Agent": "Mozilla/5.0" } });
        let html = edhResp.text;
        let rankMatch = html.match(/Rank(?:ing)?\s*#?(\d+)|#(\d+)\s*Commander/i);
        let rank = rankMatch ? parseInt(rankMatch[1] || rankMatch[2]) : 0;
        
        let themeRegex = new RegExp(`href="/commanders/${slug}/([^/"]+)"`, "g");
        let themesMatches = []; let m;
        while ((m = themeRegex.exec(html)) !== null) { themesMatches.push(m[1]); }
        const ignore = new Set(['budget', 'expensive', 'cards', 'articles', 'tournaments', 'salt', 'past-month', 'new-cards', 'combos', 'decks', 'reviews', 'sets', 'themes', 'flavor-text', 'advanced', 'year', 'month', 'week', 'exhibition', 'core', 'upgraded', 'precon', 'precons', 'high-synergy', 'recent', 'primer', 'reprints', 'cedh', 'optimized', 'casual', 'high-power', 'battlecruiser', 'bracket-1', 'bracket-2', 'bracket-3', 'bracket-4', 'bracket-5']);
        let valid_themes = [...new Set(themesMatches)].filter(t => !ignore.has(t) && t.length > 2).map(t => t.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase())).slice(0, 3);

        return { rank, name: sfData.name || name_query, color: getColorName(sfData.color_identity), themes: valid_themes, played: false, wins: 0, image_url, edhrec_url: clean_url };
    } catch (e) { return null; }
}

async function saveData() { await app.vault.adapter.write(jsonPath, JSON.stringify(data, null, 4)); }

// --- 3. UI COMPONENTS ---

const mainContainer = dv.container.createEl("div", { attr: { style: "width: 100%;" }});
const loadingMsg = mainContainer.createEl("div", { text: "⏳ Loading Commander Dashboard...", attr: { style: "padding: 20px; text-align: center; color: var(--text-muted); font-style: italic;" }});

function renderTable(container) {
    container.empty();
    data.sort((a, b) => {
        let valA = a[sortCol]; let valB = b[sortCol];
        if (typeof valA === "string") valA = valA.toLowerCase();
        return valA < valB ? (sortAsc ? -1 : 1) : (sortAsc ? 1 : -1);
    });

    const table = container.createEl("table", { attr: { class: "commander-table" }});
    const trHead = table.createEl("thead").createEl("tr");
    
    const headers = [
        { id: "rank", label: "Rank", width: "70px" },
        { id: "image_url", label: "Art", width: "80px" }, 
        { id: "name", label: "Commander", width: "200px" }, 
        { id: "color", label: "Color", width: "100px" },
        { id: "themes", label: "Themes", width: "auto" },   
        { id: "played", label: "Played", width: "90px" }, 
        { id: "wins", label: "Wins", width: "70px" },
        { id: "actions", label: "", width: "40px" } 
    ];

    headers.forEach(h => {
        const th = trHead.createEl("th", { attr: { style: h.id === 'themes' ? '' : 'width: ' + h.width + ';' }});
        if (!["image_url", "themes", "actions"].includes(h.id)) {
            th.setText(h.label + (sortCol === h.id ? (sortAsc ? " ▲" : " ▼") : ""));
            th.onclick = () => { sortAsc = sortCol === h.id ? !sortAsc : true; sortCol = h.id; renderTable(container); };
        } else th.setText(h.label);
    });

    const tbody = table.createEl("tbody");
    data.forEach((item, index) => {
        const tr = tbody.createEl("tr");
        tr.createEl("td", { text: "#" + item.rank });
        
        const tdImg = tr.createEl("td");
        if (item.image_url) {
            const img = tdImg.createEl("img", { attr: { src: item.image_url, width: "60" }});
            img.onclick = (e) => {
                e.stopPropagation();
                const overlay = document.body.createEl("div", { attr: { style: "position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.85); display:flex; justify-content:center; align-items:center; z-index:9999; cursor:zoom-out;" }});
                overlay.createEl("img", { attr: { src: item.image_url, style: "max-height:90%; border-radius:10px; animation: zoomIn 0.2s ease-out;" }});
                overlay.onclick = () => overlay.remove();
            };
        }

        const tdName = tr.createEl("td", { attr: { style: "text-align: left;" }});
        tdName.createEl("a", { text: item.name, attr: { href: item.edhrec_url, target: "_blank", style: "font-weight: bold; text-decoration: none;" }});
        
        tr.createEl("td", { text: item.color });
        tr.createEl("td", { text: (item.themes || []).join(", "), attr: { style: "font-size: 0.85em; color: var(--text-muted); text-align: left;" }});
        
        const tdPlayed = tr.createEl("td").createEl("input", { attr: { type: "checkbox", style: "cursor: pointer;" }});
        tdPlayed.checked = item.played; 
        tdPlayed.onchange = async () => { item.played = tdPlayed.checked; await saveData(); };
        
        const tdWins = tr.createEl("td").createEl("input", { attr: { type: "number", value: item.wins, style: "width: 45px; background: transparent; border: 1px solid gray; color: inherit; text-align:center; border-radius: 4px;" }});
        tdWins.onchange = async () => { item.wins = parseInt(tdWins.value) || 0; await saveData(); };

        const btnDel = tr.createEl("td").createEl("button", { text: "✕", attr: { style: "background:transparent; border:none; color:var(--text-faint); cursor:pointer; font-size: 1.1em;" }});
        btnDel.onclick = async () => { if(confirm(`Remove ${item.name}?`)) { data.splice(index, 1); await saveData(); renderTable(container); }};
    });
}

async function init() {
    try {
        const file = await app.vault.adapter.read(jsonPath);
        data = JSON.parse(file);
    } catch (e) { data = []; await app.vault.adapter.write(jsonPath, "[]"); }

    loadingMsg.remove();
    
    const formDiv = mainContainer.createEl("div", { attr: { style: "display: flex; gap: 10px; margin-bottom: 20px; align-items: center;" }});
    const inputUrl = formDiv.createEl("input", { attr: { type: "text", placeholder: "Paste EDHREC link here...", style: "flex: 1; padding: 10px; border-radius: 6px; background: var(--background-primary); border: 1px solid var(--background-modifier-border);" }});
    const btnAdd = formDiv.createEl("button", { text: "Add", attr: { style: "padding: 10px 20px; cursor: pointer; border-radius: 6px; background: var(--interactive-accent); color: white; border: none; font-weight: bold;" }});
    const statusText = formDiv.createEl("span", { attr: { style: "font-size: 0.9em; color: var(--text-muted); min-width: 120px;" }});
    
    const tableContainer = mainContainer.createEl("div", { attr: { style: "width: 100%; overflow-x: auto;" }});

    btnAdd.onclick = async () => {
        const url = inputUrl.value.trim();
        if (!url) return;
        
        // Verificação de duplicata
        const match = url.match(/\/(?:commanders|cards)\/([^/]+)/);
        if (match && data.some(c => c.edhrec_url.includes(match[1]))) {
            statusText.innerText = "Already exists!";
            statusText.style.color = "#e5c07b";
            return;
        }

        btnAdd.disabled = true;
        statusText.innerText = "Fetching...";
        statusText.style.color = "var(--text-muted)";
        
        const newData = await fetchCommanderData(url);
        if (newData) { 
            data.push(newData); 
            await saveData(); 
            renderTable(tableContainer); 
            inputUrl.value = ""; 
            statusText.innerText = "Added!";
            statusText.style.color = "#98c379";
        } else {
            statusText.innerText = "Error!";
            statusText.style.color = "#e06c75";
        }
        
        btnAdd.disabled = false;
        setTimeout(() => { statusText.innerText = ""; }, 3000);
    };

    renderTable(tableContainer);
}

init();
```