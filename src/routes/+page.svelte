<script>
    import { onMount } from 'svelte';
    import { fade, scale } from 'svelte/transition';

    /**
     * Get random cell that is not a mine (used to place mines)
     * @param {Array<Array<number>>} board
     * @param {number} startX x-coordinate of the first cell clicked
     * @param {number} startY y-coordinate of the first cell clicked
     * @returns {[number, number]} [x, y]
     */
    function getRandomSafeCell(board, startX, startY) {
        let width = board[0].length;
        let height = board.length;
        let x = Math.floor(Math.random() * width);
        let y = Math.floor(Math.random() * height);
        // redo selection if cell is a mine
        if (board[y][x] == -1) return getRandomSafeCell(board, startX, startY);
        // first-clicked cell (and cells around it) should also be safe cells
        if (Math.abs(startX - x) <= 1 && Math.abs(startY - y) <= 1) return getRandomSafeCell(board, startX, startY);
        return [x, y];
    }

    /**
     * Create array with minesweeper data
     * @param {number} width # of columns
     * @param {number} height # of rows
     * @param {number} mines # of mines
     * @param {number} startX x-coordinate of the first cell clicked
     * @param {number} startY y-coordinate of the first cell clicked
     * @returns {Array<Array<number>>} board
     */
    function createBoard(width, height, mines, startX, startY) {
        let board = [];
        for (let r = 0; r < height; r++) board.push(new Array(width).fill(0));

        for (let i = 0; i < mines; i++) {
            let [x, y] = getRandomSafeCell(board, startX, startY);
            board[y][x] = -1;
            for (let r = y - 1; r <= y + 1; r++) {
                if (r >= 0 && r < height) {
                    for (let c = x - 1; c <= x + 1; c++) {
                        if (c >= 0 && c < width) {
                            if (board[r][c] != -1) board[r][c] += 1;
                        }
                    }
                }
            }
        }

        return board;
    }

    /** @type {Array<Array<number>>} */
    let board = [[]];

    /** @type {Array<string>} */
    let squaresUncovered = [];
    /** @type {Array<string>} */
    let flags = [];

    /**
     * Reveal whole patch of empty cells
     * @param {number} x x-coordinate of the target square
     * @param {number} y y-coordinate of the target square
     */
    function revealEmptySquaresAround(x, y) {
        for (let r = y - 1; r <= y + 1; r++) {
            if (r >= 0 && r < board.length) {
                for (let c = x - 1; c <= x + 1; c++) {
                    if (c >= 0 && c < board[0].length) {
                        if (!squaresUncovered.includes(`${c},${r}`)) {
                            squaresUncovered = [...squaresUncovered, `${c},${r}`];
                            if (flags.includes(`${c},${r}`)) {
                                // remove flag
                                flags = flags.filter(flag => flag !== `${c},${r}`);
                            }
                            if (board[r][c] === 0) {
                                revealEmptySquaresAround(c, r);
                            }
                        }
                    }
                }
            }
        }
    }

    // color coding for number squares
    /** @type {Object<number, string>} */
    let colors = {
        1: "darkgreen",
        2: "darkgoldenrod",
        3: "orangered",
        4: "darkred",
        5: "darkslateblue",
        6: "darkcyan",
        7: "darkmagenta",
        8: "grey",
    };

    let boardWidth = 10;
    let boardHeight = 10;
    let numberOfMines = 20;

    onMount(() => {
        board = createBoard(boardWidth, boardHeight, numberOfMines, 0, 0);
    });
</script>

<table style:border-collapse="collapse">
    {#each board as row, y}
        <tr style:height="28px">
            {#each row as cell, x}
                <td style:width="28px" style:height="28px" style:text-align="center" style:border="1px solid grey"
                    style:background-color="black" style:box-sizing="border-box">
                    <div style:position="relative" style:width="100%" style:height="100%">
                        <div style:position="absolute" style:top="50%" style:left="50%" style:transform="translate(-50%, -50%)">
                            {#if cell > 0}
                                <b style:color={colors[cell]}>{cell}</b>
                            {:else if cell === -1}
                                <i style:color="white">&#x2022;</i>
                            {/if}
                        </div>
                        {#if !squaresUncovered.includes(`${x},${y}`)}
                            <button style:display="block" style:width="100%" style:height="100%"
                                style:position="absolute" style:top="0" style:left="0"
                                on:click={() => {
                                    if (squaresUncovered.length === 0) {
                                        // first click of the game
                                        board = createBoard(boardWidth, boardHeight, numberOfMines, x, y);
                                    }
                                    squaresUncovered = [...squaresUncovered, `${x},${y}`];
                                    if (flags.includes(`${x},${y}`)) {
                                        // remove flag
                                        flags = flags.filter(flag => flag !== `${x},${y}`);
                                    }
                                    if (board[y][x] === 0) {
                                        // reveal whole patch of empty cells
                                        revealEmptySquaresAround(x, y);
                                    }
                                }}
                                on:contextmenu={(e) => {
                                    e.preventDefault();
                                    if (flags.includes(`${x},${y}`)) {
                                        // remove flag
                                        flags = flags.filter(flag => flag !== `${x},${y}`);
                                    } else {
                                        // add flag
                                        flags = [...flags, `${x},${y}`];
                                    }
                                }} transition:scale>
                                {#if flags.includes(`${x},${y}`)}
                                    <span style:display="inline-block" transition:scale={{ duration: 160, }}>&#x2691;</span>
                                {/if}
                            </button>
                        {/if}
                    </div>
                </td>
            {/each}
        </tr>
    {/each}
</table>
<span>&#x2691;</span>: {flags.length} / {numberOfMines}
