# Shaxmat-arena

<!DOCTYPE html>
<html lang="uz">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<meta name="theme-color" content="#111827">
<title>SHAXMAT ARENA</title>

<script src="https://telegram.org/js/telegram-web-app.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js"></script>

<style>
* { box-sizing:border-box; }
body {
    margin:0;
    padding:16px;
    background:#111827;
    color:#fff;
    font-family:Arial,sans-serif;
    user-select:none;
}
.header {
    display:flex;
    justify-content:space-between;
    align-items:center;
    margin-bottom:20px;
}
.logo { font-size:21px; font-weight:bold; }
.logo span { color:#d6a84f; }
.badge {
    background:#263449;
    padding:7px 12px;
    border-radius:20px;
    color:#d6a84f;
    font-size:12px;
}
.panel {
    background:#1f2937;
    padding:13px;
    border-radius:12px;
    margin-bottom:13px;
}
.player {
    display:flex;
    justify-content:space-between;
    align-items:center;
}
.player-name { font-weight:bold; }
.clock {
    background:#080e19;
    padding:8px 15px;
    border-radius:8px;
    font-size:20px;
    font-weight:bold;
    min-width:75px;
    text-align:center;
}
.clock.active { background:#d6a84f; color:#111827; }
.board-wrap {
    width:100%;
    max-width:500px;
    margin:12px auto;
}
.board {
    width:100%;
    aspect-ratio:1;
    display:grid;
    grid-template-columns:repeat(8,1fr);
    border:4px solid #293548;
    border-radius:5px;
    overflow:hidden;
}
.square {
    position:relative;
    display:flex;
    align-items:center;
    justify-content:center;
    aspect-ratio:1;
    font-size:clamp(27px,8vw,48px);
    cursor:pointer;
}
.light { background:#e9dfc7; }
.dark { background:#77936c; }
.white-piece {
    color:#fff;
    text-shadow:1px 2px 2px #222;
}
.black-piece {
    color:#171717;
    text-shadow:0 1px 1px #aaa;
}
.selected { background:#e6c34f !important; }
.target::after {
    content:"";
    width:25%;
    height:25%;
    border-radius:50%;
    background:#263e35aa;
    position:absolute;
}
.capture::after {
    content:"";
    position:absolute;
    inset:3px;
    border:4px solid #b34d42;
    border-radius:50%;
}
.coordinate {
    position:absolute;
    bottom:2px;
    left:3px;
    font-size:9px;
    opacity:.65;
    color:#222;
}
.status {
    text-align:center;
    font-size:14px;
    color:#d6a84f;
    min-height:22px;
    margin:12px 0;
}
.buttons {
    display:grid;
    grid-template-columns:repeat(3,1fr);
    gap:8px;
    margin-top:12px;
}
button {
    border:0;
    border-radius:9px;
    padding:13px 5px;
    background:#d6a84f;
    color:#171717;
    font-weight:bold;
    font-size:13px;
}
button.secondary {
    background:#293548;
    color:white;
}
.history {
    max-height:110px;
    overflow:auto;
    font-size:13px;
    line-height:1.8;
    color:#d1d5db;
}
.footer {
    text-align:center;
    color:#718096;
    font-size:11px;
    margin-top:18px;
}
</style>
</head>

<body>

<div class="header">
    <div class="logo">♟ SHAXMAT <span>ARENA</span></div>
    <div class="badge">BETA</div>
</div>

<div class="panel player">
    <div>
        <div class="player-name" id="blackName">⚫ Qora</div>
        <small>Qora donalar</small>
    </div>
    <div class="clock" id="blackClock">10:00</div>
</div>

<div class="board-wrap">
    <div class="board" id="board"></div>
</div>

<div class="panel player">
    <div>
        <div class="player-name" id="whiteName">⚪ Oq</div>
        <small>Oq donalar</small>
    </div>
    <div class="clock active" id="whiteClock">10:00</div>
</div>

<div class="status" id="status">Oq donalar yuradi</div>

<div class="buttons">
    <button onclick="newGame()">Yangi o‘yin</button>
    <button class="secondary" onclick="undoMove()">Ortga</button>
    <button class="secondary" onclick="flipBoard()">Aylantirish</button>
</div>

<div class="panel" style="margin-top:15px">
    <b>Yurishlar tarixi</b>
    <div class="history" id="history">Hali yurish yo‘q</div>
</div>

<div class="footer">
    SHAXMAT ARENA • BETA VERSION
</div>

<script>
const tg = window.Telegram?.WebApp;
if (tg) {
    tg.ready();
    tg.expand();
}

const game = new Chess();

const pieces = {
    wk:"♔", wq:"♕", wr:"♖", wb:"♗", wn:"♘", wp:"♙",
    bk:"♚", bq:"♛", br:"♜", bb:"♝", bn:"♞", bp:"♟"
};

let selected = null;
let legalMoves = [];
let flipped = false;
let whiteTime = 600;
let blackTime = 600;
let clockRunning = true;
let lastTime = Date.now();

const boardElement = document.getElementById("board");

function renderBoard() {
    boardElement.innerHTML = "";

    const files = flipped
        ? ["h","g","f","e","d","c","b","a"]
        : ["a","b","c","d","e","f","g","h"];

    const ranks = flipped
        ? [1,2,3,4,5,6,7,8]
        : [8,7,6,5,4,3,2,1];

    ranks.forEach((rank, row) => {
        files.forEach((file, col) => {
            const squareName = file + rank;
            const piece = game.get(squareName);

            const square = document.createElement("div");
            square.className = "square " +
                ((row + col) % 2 === 0 ? "light" : "dark");

            if (selected === squareName) {
                square.classList.add("selected");
            }

            const move = legalMoves.find(m => m.to === squareName);

            if (move) {
                square.classList.add(
                    piece ? "capture" : "target"
                );
            }

            if (piece) {
                const span = document.createElement("span");
                span.textContent = pieces[piece.color + piece.type];
                span.className =
                    piece.color === "w" ? "white-piece" : "black-piece";
                square.appendChild(span);
            }

            if (col === 0) {
                const coordinate = document.createElement("small");
                coordinate.className = "coordinate";
                coordinate.textContent = rank;
                square.appendChild(coordinate);
            }

            square.addEventListener("click", () => {
                clickSquare(squareName);
            });

            boardElement.appendChild(square);
        });
    });

    updateStatus();
    updateHistory();
}

function clickSquare(square) {
    if (game.game_over()) return;

    const piece = game.get(square);

    if (selected) {
        const move = legalMoves.find(m => m.to === square);

        if (move) {
            let promotion = "q";

            if (move.flags.includes("p")) {
                const answer = prompt(
                    "Donani tanlang: q - Farzin, r - Rux, b - Fil, n - Ot",
                    "q"
                );
                promotion = ["q","r","b","n"].includes(answer)
                    ? answer : "q";
            }

            try {
                game.move({
                    from: selected,
                    to: square,
                    promotion: promotion
                });
            } catch (e) {}

            selected = null;
            legalMoves = [];
            renderBoard();
            return;
        }
    }

    if (piece && piece.color === game.turn()) {
        selected = square;
        legalMoves = game.moves({
            square: square,
            verbose: true
        });
    } else {
        selected = null;
        legalMoves = [];
    }

    renderBoard();
}

function updateStatus() {
    const status = document.getElementById("status");

    if (game.in_checkmate()) {
        status.textContent =
            "♚ Shoh mot! " +
            (game.turn() === "w" ? "Qora" : "Oq") +
            " g‘alaba qozondi!";
    } else if (game.in_draw()) {
        status.textContent = "Durang!";
    } else if (game.in_check()) {
        status.textContent = "Shohga shax!";
    } else {
        status.textContent =
            game.turn() === "w" ? "Oq donalar yuradi" : "Qora donalar yuradi";
    }
}

function updateHistory() {
    const history = game.history();

    document.getElementById("history").textContent =
        history.length
            ? history.map((m, i) =>
                (i % 2 === 0 ? (Math.floor(i / 2) + 1) + ". " : "") + m
              ).join(" ")
            : "Hali yurish yo‘q";
}

function newGame() {
    game.reset();
    selected = null;
    legalMoves = [];
    whiteTime = 600;
    blackTime = 600;
    clockRunning = true;
    lastTime = Date.now();
    renderBoard();
    updateClocks();
}

function undoMove() {
    game.undo();
    selected = null;
    legalMoves = [];
    renderBoard();
}

function flipBoard() {
    flipped = !flipped;
    renderBoard();
}

function formatTime(seconds) {
    const s = Math.max(0, seconds);
    const m = Math.floor(s / 60);
    const r = s % 60;
    return String(m).padStart(2,"0") + ":" +
           String(r).padStart(2,"0");
}

function updateClocks() {
    document.getElementById("whiteClock").textContent =
        formatTime(whiteTime);

    document.getElementById("blackClock").textContent =
        formatTime(blackTime);

    document.getElementById("whiteClock").classList.toggle(
        "active", game.turn() === "w" && !game.game_over()
    );

    document.getElementById("blackClock").classList.toggle(
        "active", game.turn() === "b" && !game.game_over()
    );
}

setInterval(() => {
    if (!clockRunning || game.game_over()) return;

    const now = Date.now();
    const elapsed = Math.floor((now - lastTime) / 1000);

    if (elapsed < 1) return;

    lastTime += elapsed * 1000;

    if (game.turn() === "w") {
        whiteTime -= elapsed;
    } else {
        blackTime -= elapsed;
    }

    if (whiteTime <= 0 || blackTime <= 0) {
        clockRunning = false;
        document.getElementById("status").textContent =
            whiteTime <= 0 ? "Vaqt tugadi! Qora g‘alaba!" :
                             "Vaqt tugadi! Oq g‘alaba!";
    }

    updateClocks();
}, 250);

renderBoard();
updateClocks();
</script>

</body>
</html>
