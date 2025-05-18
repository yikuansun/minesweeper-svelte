<script>
    import { onMount } from 'svelte';
    import { get } from 'svelte/store';

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
        // first-clicked cell should also be a safe cell
        if (startX == x && startY == y) return getRandomSafeCell(board, startX, startY);
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

    // color coding for number squares
    let colors = {
        1: "darkgreen",
        2: "darkgoldenrod",
        3: "orangered",
        4: "darkred",
        5: "darkslateblue",
        6: "darkcyan",
        7: "darkmagenta",
        8: "dimgrey",
    };

    onMount(() => {
        board = createBoard(10, 10, 10, 0, 0);
    });
</script>

<table style:border-collapse="collapse">
    {#each board as row, y}
        <tr style:height="28px">
            {#each row as cell, x}
                <td style:width="28px" style:text-align="center" style:border="1px solid grey">
                    {#if cell > 0}
                        <b style:color={colors[cell]}>{cell}</b>
                    {:else if cell === -1}
                        M
                    {/if}
                </td>
            {/each}
        </tr>
    {/each}
</table>