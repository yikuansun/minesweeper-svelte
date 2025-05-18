<script>
    import { onMount } from 'svelte';

    /**
     * Create array with minesweeper data
     * @param {number} width # of columns
     * @param {number} height # of rows
     * @param {number} mines # of mines
     * @returns {Array<Array<number>>} board
     */
    function createBoard(width, height, mines) {
        let board = [];
        for (let r = 0; r < height; r++) board.push(new Array(width).fill(0));

        for (let i = 0; i < mines; i++) {
            let x = Math.floor(Math.random() * width);
            let y = Math.floor(Math.random() * height);
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

    let boardWidth = 10;
    let boardHeight = 10;
    /** @type {Array<Array<number>>} */
    let board = [[]];

    onMount(() => {
        createBoard(10, 10, 10);
        board = createBoard(10, 10, 10);
    });
</script>

<table style:border-collapse="collapse">
    {#each board as row, y}
        <tr style:height="28px">
            {#each row as cell, x}
                <td style:width="28px" style:text-align="center" style:border="1px solid grey">
                    {#if cell > 0}
                        <b>{cell}</b>
                    {:else if cell === -1}
                        M
                    {/if}
                </td>
            {/each}
        </tr>
    {/each}
</table>