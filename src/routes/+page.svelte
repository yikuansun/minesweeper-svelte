<script>
    import { onMount } from 'svelte';
    import { fade, scale } from 'svelte/transition';
    import clickSound from "$lib/sounds/click.wav";
    import explodeSound from "$lib/sounds/explode.wav";
    import winSound from "$lib/sounds/win.wav";
    import squaresSound from "$lib/sounds/squares.wav";
    import flagSound from "$lib/sounds/flag.wav";

    /**
     * Play sound effect
     * @param {string} sound path of the sound file
     */
    function playSound(sound) {
        let aud = new Audio(sound);
        aud.play();
    }

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

    let boardWidth = 12;
    let boardHeight = 12;
    let numberOfMines = 24;

    let loser = false;
    let winner = false;

    let startTime = 0;
    let currentTime = 0;
    let timerInterval = 0;

    /** @type {HTMLDivElement} */
    let gameDisplay;
    let displayScale = 2;
    function getDisplayScale() {
        displayScale = Math.min(
            window.innerWidth / gameDisplay.clientWidth,
            window.innerHeight / gameDisplay.clientHeight,
        );
        displayScale -= 0.2; // pad a bit
    }

    let startScreenVisible = true;

    onMount(() => {
        board = createBoard(boardWidth, boardHeight, 0, 0, 0);
        setTimeout(getDisplayScale, 1);
        window.addEventListener("resize", getDisplayScale);
    });
</script>

<div style:position="fixed" style:top="50%" style:left="50%"
    style:transform="translate(-50%, -50%) scale({displayScale})" style:color="lightgrey"
    bind:this={gameDisplay}>
    <table style:width="100%">
        <tr>
            <td style:text-align="left">
                <span>&#x2691;</span>: {flags.length} / {numberOfMines}
            </td>
            <td style:text-align="right">
                {(new Date(currentTime - startTime)).getMinutes()}
                :
                {((new Date(currentTime - startTime)).getSeconds() + "").padStart(2, "0")}
                .
                {Math.floor((new Date(currentTime - startTime)).getMilliseconds() / 100)}
            </td>
        </tr>
    </table>

    <table style:border-collapse="collapse">
        {#each board as row, y}
            <tr style:height="28px">
                {#each row as cell, x}
                    <td style:text-align="center" style:border="1px solid grey"
                        style:background-color="black" style:box-sizing="border-box">
                        <div style:position="relative" style:width="25px" style:height="25px">
                            <div style:position="absolute" style:top="50%" style:left="50%" style:transform="translate(-50%, -50%)">
                                {#if cell > 0}
                                    <b style:color={colors[cell]}>{cell}</b>
                                {:else if cell === -1}
                                    &#128165;
                                {/if}
                            </div>
                            {#if !squaresUncovered.includes(`${x},${y}`)}
                                <button style:display="block" style:width="100%" style:height="100%"
                                    style:position="absolute" style:top="0" style:left="0"
                                    on:click={() => {
                                        if (loser) return; // can't play after losing
                                        if (flags.includes(`${x},${y}`)) {
                                            // don't allow clicking on flagged squares
                                            return;
                                        }
                                        if (squaresUncovered.length === 0) {
                                            // first click of the game
                                            board = createBoard(boardWidth, boardHeight, numberOfMines, x, y);
                                            // start in-game timer
                                            startTime = Date.now();
                                            currentTime = startTime;
                                            timerInterval = setInterval(() => {
                                                currentTime = Date.now();
                                            }, 100);
                                        }
                                        squaresUncovered = [...squaresUncovered, `${x},${y}`];
                                        playSound(squaresSound);
                                        if (board[y][x] === 0) {
                                            // reveal whole patch of empty cells
                                            revealEmptySquaresAround(x, y);
                                        }
                                        else if (board[y][x] === -1) {
                                            // hit a mine :(((
                                            loser = true;
                                            clearInterval(timerInterval);
                                            playSound(explodeSound);
                                        }
                                        if (squaresUncovered.length >= (boardWidth * boardHeight - numberOfMines) && !loser) {
                                            // win condition
                                            winner = true;
                                            clearInterval(timerInterval);
                                            playSound(winSound);
                                        }
                                    }}
                                    on:contextmenu={(e) => {
                                        e.preventDefault();
                                        if (loser) return; // can't play after losing
                                        if (flags.includes(`${x},${y}`)) {
                                            // remove flag
                                            flags = flags.filter(flag => flag !== `${x},${y}`);
                                        } else {
                                            // add flag
                                            flags = [...flags, `${x},${y}`];
                                        }
                                        playSound(flagSound);
                                    }} transition:scale
                                    disabled={loser} style:transition="opacity 1s"
                                    style:background-color="grey" style:color="darkred" style:border="3px inset silver"
                                    style:opacity={loser ? 0.5 : 1}>
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
{#if loser}
    <div style:position="absolute" style:top="50%" style:left="50%"
        style:transform="translate(-50%, -50%)" style:background-color="#141414"
        style:padding="5px" style:color="white" style:border-radius="5px" style:box-shadow="0 0 10px black"
        style:text-align="center"
        transition:fade={{ delay: 1000, duration: 500, }}>
        You suck at this game. Try again.
    </div>
{/if}
{#if winner}
    <div style:position="absolute" style:top="50%" style:left="50%"
        style:transform="translate(-50%, -50%)" style:background-color="#141414"
        style:padding="5px" style:color="white" style:border-radius="5px" style:box-shadow="0 0 10px black"
        style:text-align="center"
        transition:fade={{ delay: 1000, duration: 500, }}>
        You win! <br />
        Final time: 
        {(new Date(currentTime - startTime)).getMinutes()}:{((new Date(currentTime - startTime)).getSeconds() + "").padStart(2, "0")}.{Math.floor((new Date(currentTime - startTime)).getMilliseconds() / 100)}
    </div>
{/if}
</div>

{#if startScreenVisible}
    <div style:position="fixed" style:top="0" style:left="0" style:width="100vw" style:height="100vh"
        style:background-color="#141414" style:z-index="100" transition:fade={{ delay: 200, duration: 500, }} style:color="white"
        class="startScreen">
        <div style:position="absolute" style:top="50%" style:left="50%" style:transform="translate(-50%, -50%)" style:text-align="center">
            <b style:font-size="50px">Minesweeper</b>
            <br /><br />
            <b style:font-size="25px">Choose a game:</b>
            <br /><br />
            <button on:click={() => {
                boardWidth = 8;
                boardHeight = 8;
                numberOfMines = 10;
                playSound(clickSound);
            }}>Beginner</button> <br />
            <button on:click={() => {
                boardWidth = 12;
                boardHeight = 12;
                numberOfMines = 24;
                playSound(clickSound);
            }}>Intermediate</button> <br />
            <button on:click={() => {
                boardWidth = 16;
                boardHeight = 16;
                numberOfMines = 50;
                playSound(clickSound);
            }}>Expert</button>
            <br /><br />
            <label>
                Width:
                <input type="number" bind:value={boardWidth} min="8" max="16" style:width="50px" />
            </label> |
            <label>
                Height:
                <input type="number" bind:value={boardHeight} min="8" max="16" style:width="50px" />
            </label> |
            <label>
                Mines:
                <input type="number" bind:value={numberOfMines} min="10" max={boardWidth * boardHeight - 9} style:width="50px" />
            </label>
            <br /><br />
            <button style:font-size="24px" on:click={() => {
                board = createBoard(boardWidth, boardHeight, numberOfMines, 0, 0);
                setTimeout(getDisplayScale, 1);
                startScreenVisible = false;
                playSound(clickSound);
            }}>Start Game</button>
        </div>
    </div>
{/if}

<style>
    :global(body) {
        background-color: #141414;
        user-select: none;
        font-family:'Courier New', Courier, monospace;
    }

    .startScreen button {
        background-color: grey;
        color: darkred;
        font-weight: bold;
        border: 7px inset silver;
        font-size: 16px;
        padding: 10px 15px;
        margin: 5px;
        font-family:'Courier New', Courier, monospace;
    }

    .startScreen button:active {
        border: 7px outset silver;
        background-color: dimgrey;
    }

    .startScreen input {
        background-color: #111111;
        color: white;
        border: 5px inset grey;
        font-size: 16px;
        padding: 7px 11px;
        margin: 3px;
        outline: none!important;
        font-family:'Courier New', Courier, monospace;
    }
</style>

<svelte:head>
    <title>Minesweeper</title>
    <meta name="author" content="Yikuan Sun" />
    <meta name="description" content="A minesweeper game made with Svelte" />
    <meta name="keywords" content="minesweeper, svelte, game, html, css, javascript" />
</svelte:head>

<svelte:body on:contextmenu={e => e.preventDefault()} />