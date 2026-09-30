<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Ajedrez</title>

<style>
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    min-height: 100vh;
    background: #111827;
    color: white;
    font-family: Arial, sans-serif;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 15px;
}

.game {
    width: min(95vw, 850px);
    text-align: center;
}

h1 {
    margin-bottom: 12px;
    font-size: clamp(25px, 6vw, 42px);
}

#status {
    margin-bottom: 12px;
    font-size: 18px;
    font-weight: bold;
}

.board {
    width: min(92vw, 720px);
    aspect-ratio: 1;
    margin: auto;
    display: grid;
    grid-template-columns: repeat(8, 1fr);
    border: 4px solid #222;
    box-shadow: 0 10px 35px #0008;
    touch-action: manipulation;
}

.square {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    user-select: none;
}

.light {
    background: #f0d9b5;
}

.dark {
    background: #b58863;
}

.square.selected {
    outline: 5px solid #ffd700;
    outline-offset: -5px;
}

.square.move::after {
    content: "";
    width: 25%;
    height: 25%;
    border-radius: 50%;
    background: #35c759;
    position: absolute;
}

.square.capture::after {
    content: "";
    width: 75%;
    height: 75%;
    border-radius: 50%;
    border: 5px solid #ff4444;
    position: absolute;
}

.piece {
    font-size: clamp(32px, 9vw, 68px);
    line-height: 1;
    z-index: 2;
    cursor: pointer;
}

.controls {
    margin-top: 15px;
    display: flex;
    justify-content: center;
    gap: 10px;
    flex-wrap: wrap;
}

button {
    border: none;
    padding: 12px 20px;
    border-radius: 10px;
    background: #2563eb;
    color: white;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
}

button:active {
    transform: scale(.96);
}

.info {
    margin-top: 15px;
    background: #1f2937;
    border-radius: 12px;
    padding: 12px;
}

#moves {
    margin-top: 8px;
    max-height: 130px;
    overflow-y: auto;
    font-size: 14px;
    text-align: left;
}

.promotion {
    position: fixed;
    inset: 0;
    background: #0009;
    display: none;
    align-items: center;
    justify-content: center;
    z-index: 10;
}

.promotion-box {
    background: #1f2937;
    padding: 20px;
    border-radius: 15px;
    text-align: center;
}

.promotion-box h2 {
    margin-bottom: 15px;
}

.promotion button {
    font-size: 35px;
    padding: 8px 15px;
    margin: 5px;
}

@media (max-width: 450px) {
    body {
        padding: 8px;
    }

    .board {
        width: 96vw;
        border-width: 2px;
    }

    .info {
        font-size: 14px;
    }

    button {
        padding: 10px 14px;
    }
}
</style>
</head>

<body>

<div class="game">

    <h1>♟️ AJEDREZ ♟️</h1>

    <div id="status">Turno: Blancas</div>

    <div id="board" class="board"></div>

    <div class="controls">
        <button onclick="restartGame()">🔄 Reiniciar</button>
        <button onclick="undoMove()">↩️ Deshacer</button>
    </div>

    <div class="info">
        <strong>Movimientos</strong>
        <div id="moves"></div>
    </div>

</div>

<div id="promotion" class="promotion">
    <div class="promotion-box">
        <h2>Elige una pieza</h2>
        <div>
            <button onclick="promote('Q')">♕</button>
            <button onclick="promote('R')">♖</button>
            <button onclick="promote('B')">♗</button>
            <button onclick="promote('N')">♘</button>
        </div>
    </div>
</div>

<script>

const boardElement = document.getElementById("board");
const statusElement = document.getElementById("status");
const movesElement = document.getElementById("moves");
const promotionElement = document.getElementById("promotion");

const symbols = {
    w: {
        K: "♔",
        Q: "♕",
        R: "♖",
        B: "♗",
        N: "♘",
        P: "♙"
    },
    b: {
        K: "♚",
        Q: "♛",
        R: "♜",
        B: "♝",
        N: "♞",
        P: "♟"
    }
};

let board;
let turn;
let selected;
let legalMoves;
let history;
let moveList;
let pendingPromotion = null;

function createInitialBoard() {

    return [
        [
            {c:"b",t:"R"},
            {c:"b",t:"N"},
            {c:"b",t:"B"},
            {c:"b",t:"Q"},
            {c:"b",t:"K"},
            {c:"b",t:"B"},
            {c:"b",t:"N"},
            {c:"b",t:"R"}
        ],

        Array(8).fill(null).map(() => ({c:"b",t:"P"})),

        Array(8).fill(null),
        Array(8).fill(null),
        Array(8).fill(null),
        Array(8).fill(null),

        Array(8).fill(null).map(() => ({c:"w",t:"P"})),

        [
            {c:"w",t:"R"},
            {c:"w",t:"N"},
            {c:"w",t:"B"},
            {c:"w",t:"Q"},
            {c:"w",t:"K"},
            {c:"w",t:"B"},
            {c:"w",t:"N"},
            {c:"w",t:"R"}
        ]
    ];
}

function restartGame() {

    board = createInitialBoard();

    turn = "w";

    selected = null;

    legalMoves = [];

    history = [];

    moveList = [];

    pendingPromotion = null;

    promotionElement.style.display = "none";

    render();

    updateStatus();
}

function render() {

    boardElement.innerHTML = "";

    for (let r = 0; r < 8; r++) {

        for (let c = 0; c < 8; c++) {

            const square = document.createElement("div");

            square.className =
                "square " +
                ((r + c) % 2 === 0 ? "light" : "dark");

            square.dataset.row = r;
            square.dataset.col = c;

            if (
                selected &&
                selected.r === r &&
                selected.c === c
            ) {
                square.classList.add("selected");
            }

            const possible = legalMoves.find(
                m => m.r === r && m.c === c
            );

            if (possible) {

                if (board[r][c]) {
                    square.classList.add("capture");
                } else {
                    square.classList.add("move");
                }
            }

            const piece = board[r][c];

            if (piece) {

                const element = document.createElement("div");

                element.className = "piece";

                element.textContent =
                    symbols[piece.c][piece.t];

                square.appendChild(element);
            }

            square.addEventListener("click", () => {
                handleSquareClick(r, c);
            });

            boardElement.appendChild(square);
        }
    }

    movesElement.innerHTML =
        moveList.map((m,i) =>
            `${i + 1}. ${m}`
        ).join("<br>");
}

function handleSquareClick(r, c) {

    if (pendingPromotion) return;

    const piece = board[r][c];

    if (selected) {

        const move = legalMoves.find(
            m => m.r === r && m.c === c
        );

        if (move) {

            makeMove(
                selected.r,
                selected.c,
                r,
                c
            );

            return;
        }
    }

    if (piece && piece.c === turn) {

        selected = {r, c};

        legalMoves = getLegalMoves(r, c);

        render();
    }
}

function getLegalMoves(r, c) {

    const piece = board[r][c];

    if (!piece) return [];

    let moves = [];

    if (piece.t === "P") {

        const direction =
            piece.c === "w" ? -1 : 1;

        const startRow =
            piece.c === "w" ? 6 : 1;

        const one = r + direction;

        if (
            one >= 0 &&
            one < 8 &&
            !board[one][c]
        ) {

            moves.push({r:one,c});

            const two = r + direction * 2;

            if (
                r === startRow &&
                !board[two][c]
            ) {
                moves.push({r:two,c});
            }
        }

        for (const dc of [-1,1]) {

            const nr = r + direction;
            const nc = c + dc;

            if (
                nr >= 0 &&
                nr < 8 &&
                nc >= 0 &&
                nc < 8 &&
                board[nr][nc] &&
                board[nr][nc].c !== piece.c
            ) {
                moves.push({r:nr,c:nc});
            }
        }
    }

    if (piece.t === "N") {

        const jumps = [
            [-2,-1],
            [-2,1],
            [-1,-2],
            [-1,2],
            [1,-2],
            [1,2],
            [2,-1],
            [2,1]
        ];

        for (const [dr,dc] of jumps) {

            addIfValid(
                moves,
                r + dr,
                c + dc,
                piece.c
            );
        }
    }

    if (
        piece.t === "B" ||
        piece.t === "R" ||
        piece.t === "Q"
    ) {

        const directions = [];

        if (
            piece.t === "B" ||
            piece.t === "Q"
        ) {
            directions.push(
                [-1,-1],
                [-1,1],
                [1,-1],
                [1,1]
            );
        }

        if (
            piece.t === "R" ||
            piece.t === "Q"
        ) {
            directions.push(
                [-1,0],
                [1,0],
                [0,-1],
                [0,1]
            );
        }

        for (const [dr,dc] of directions) {

            let nr = r + dr;
            let nc = c + dc;

            while (
                nr >= 0 &&
                nr < 8 &&
                nc >= 0 &&
                nc < 8
            ) {

                if (!board[nr][nc]) {

                    moves.push({r:nr,c:nc});

                } else {

                    if (board[nr][nc].c !== piece.c) {
                        moves.push({r:nr,c:nc});
                    }

                    break;
                }

                nr += dr;
                nc += dc;
            }
        }
    }

    if (piece.t === "K") {

        for (let dr = -1; dr <= 1; dr++) {

            for (let dc = -1; dc <= 1; dc++) {

                if (dr === 0 && dc === 0) continue;

                addIfValid(
                    moves,
                    r + dr,
                    c + dc,
                    piece.c
                );
            }
        }
    }

    return moves.filter(move => {

        const copy = cloneBoard(board);

        copy[move.r][move.c] =
            copy[r][c];

        copy[r][c] = null;

        return !isKingInCheck(copy, piece.c);
    });
}

function addIfValid(moves,r,c,color) {

    if (
        r < 0 ||
        r >= 8 ||
        c < 0 ||
        c >= 8
    ) return;

    if (
        !board[r][c] ||
        board[r][c].c !== color
    ) {
        moves.push({r,c});
    }
}

function makeMove(fr,fc,tr,tc) {

    history.push({
        board: cloneBoard(board),
        turn: turn,
        moveList: [...moveList]
    });

    const piece = board[fr][fc];

    const captured = board[tr][tc];

    board[tr][tc] = piece;
    board[fr][fc] = null;

    let notation =
        symbols[piece.c][piece.t] +
        " " +
        squareName(tr,tc);

    if (captured) {
        notation += " × " +
            symbols[captured.c][captured.t];
    }

    moveList.push(notation);

    selected = null;
    legalMoves = [];

    if (
        piece.t === "P" &&
        (tr === 0 || tr === 7)
    ) {

        pendingPromotion = {
            r: tr,
            c: tc
        };

        promotionElement.style.display = "flex";

        render();

        return;
    }

    turn = turn === "w" ? "b" : "w";

    render();

    updateStatus();
}

function promote(type) {

    if (!pendingPromotion) return;

    board[pendingPromotion.r][pendingPromotion.c] = {
        c: turn === "w" ? "b" : "w",
        t: type
    };

    pendingPromotion = null;

    promotionElement.style.display = "none";

    turn = turn === "w" ? "b" : "w";

    render();

    updateStatus();
}

function squareName(r,c) {

    const letters = "abcdefgh";

    return letters[c] + (8-r);
}

function cloneBoard(b) {

    return b.map(row =>
        row.map(piece =>
            piece ? {...piece} : null
        )
    );
}

function findKing(b,color) {

    for (let r=0;r<8;r++) {

        for (let c=0;c<8;c++) {

            if (
                b[r][c] &&
                b[r][c].c === color &&
                b[r][c].t === "K"
            ) {
                return {r,c};
            }
        }
    }

    return null;
}

function isKingInCheck(b,color) {

    const king = findKing(b,color);

    if (!king) return true;

    const enemy =
        color === "w" ? "b" : "w";

    for (let r=0;r<8;r++) {

        for (let c=0;c<8;c++) {

            const piece = b[r][c];

            if (!piece || piece.c !== enemy)
                continue;

            if (
                attacksSquare(
                    b,
                    r,
                    c,
                    king.r,
                    king.c
                )
            ) {
                return true;
            }
        }
    }

    return false;
}

function attacksSquare(
    b,
    r,
    c,
    tr,
    tc
) {

    const piece = b[r][c];

    if (piece.t === "P") {

        const direction =
            piece.c === "w" ? -1 : 1;

        return (
            tr === r + direction &&
            Math.abs(tc-c) === 1
        );
    }

    if (piece.t === "N") {

        const dr = Math.abs(tr-r);
        const dc = Math.abs(tc-c);

        return (
            (dr === 2 && dc === 1) ||
            (dr === 1 && dc === 2)
        );
    }

    if (piece.t === "K") {

        return (
            Math.abs(tr-r) <= 1 &&
            Math.abs(tc-c) <= 1
        );
    }

    let dr = Math.sign(tr-r);
    let dc = Math.sign(tc-c);

    const validBishop =
        Math.abs(tr-r) === Math.abs(tc-c);

    const validRook =
        tr === r || tc === c;

    if (piece.t === "B" && !validBishop)
        return false;

    if (piece.t === "R" && !validRook)
        return false;

    if (
        piece.t === "Q" &&
        !validBishop &&
        !validRook
    )
        return false;

    let nr = r + dr;
    let nc = c + dc;

    while (nr !== tr || nc !== tc) {

        if (b[nr][nc]) return false;

        nr += dr;
        nc += dc;
    }

    return true;
}

function hasAnyLegalMove(color) {

    for (let r=0;r<8;r++) {

        for (let c=0;c<8;c++) {

            if (
                board[r][c] &&
                board[r][c].c === color
            ) {

                if (
                    getLegalMoves(r,c).length > 0
                ) {
                    return true;
                }
            }
        }
    }

    return false;
}

function updateStatus() {

    const name =
        turn === "w"
        ? "Blancas"
        : "Negras";

    if (isKingInCheck(board,turn)) {

        if (!hasAnyLegalMove(turn)) {

            statusElement.textContent =
                "♚ JAQUE MATE — " +
                (turn === "w"
                    ? "Ganan negras"
                    : "Ganan blancas");

            return;
        }

        statusElement.textContent =
            "⚠️ JAQUE — Turno: " + name;

        return;
    }

    if (!hasAnyLegalMove(turn)) {

        statusElement.textContent =
            "🤝 TABLAS — Ahogado";

        return;
    }

    statusElement.textContent =
        "Turno: " + name;
}

function undoMove() {

    if (history.length === 0) return;

    const previous =
        history.pop();

    board =
        cloneBoard(previous.board);

    turn =
        previous.turn;

    moveList =
        [...previous.moveList];

    selected = null;

    legalMoves = [];

    pendingPromotion = null;

    promotionElement.style.display = "none";

    render();

    updateStatus();
}

restartGame();

</script>

</body>
</html>