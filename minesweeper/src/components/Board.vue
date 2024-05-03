<script setup>
import { ref } from 'vue'

const STATUS = {
    READY: 'ready',
    PLAYING: 'playing',
    WON: 'won',
    LOST: 'lost'
}

const gameStatus = ref(STATUS.READY)

const board = ref(Array.from({ length: 20 }, (_, row) => Array.from({ length: 23 }, (_, col) => ({
    row,
    col,
    isMine: false,
    isRevealed: false,
    isFlagged: false,
    adjacentMines: 0,
    highlight: false
}))))

const cellAsChar = (cell) => {
    if (cell.isMine) {
        return '*'
    } else {
        return cell.adjacentMines == 0 ? '-' : cell.adjacentMines
    }
}

const setupBoard = () => {  
    const mines = 99
    const rows = 20
    const cols = 23

    // reset
    for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
            board.value[row][col].isMine = false
            board.value[row][col].isRevealed = false
            board.value[row][col].isFlagged = false
            board.value[row][col].adjacentMines = 0
        }
    }

    for (let i = 0; i < mines; i++) {
        let row = Math.floor(Math.random() * rows)
        let col = Math.floor(Math.random() * cols)

        if (board.value[row][col].isMine) {
            i--
        } else {
            board.value[row][col].isMine = true
        }
        console.log('OK', row, col)
    }

    for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
            if (board.value[row][col].isMine) {
                board.value[row][col].adjacentMines = -1
                continue
            }

            let count = 0
            for (let i = -1; i <= 1; i++) {
                for (let j = -1; j <= 1; j++) {
                    if (i === 0 && j === 0) {
                        continue
                    }

                    let newRow = row + i
                    let newCol = col + j

                    if (newRow < 0 || newRow >= rows || newCol < 0 || newCol >= cols) {
                        continue
                    }

                    if (board.value[newRow][newCol].isMine) {
                        count++
                    }
                }
            }

            board.value[row][col].adjacentMines = count
        }
    }
}

const reveal = (row, col) => {
    if (row < 0 || row >= 20 || col < 0 || col >= 23) {
        return
    }

    if (board.value[row][col].isRevealed) {
        return
    }

    if (board.value[row][col].isFlagged) {
        return
    }

    if (board.value[row][col].isMine) {
        gameStatus.value = STATUS.LOST
        board.value[row][col].isRevealed = true
        return
    }

    board.value[row][col].isRevealed = true

    if (board.value[row][col].adjacentMines === 0) {
        reveal(row - 1, col - 1)
        reveal(row - 1, col)
        reveal(row - 1, col + 1)
        reveal(row, col - 1)
        reveal(row, col + 1)
        reveal(row + 1, col - 1)
        reveal(row + 1, col)
        reveal(row + 1, col + 1)
    }
}

const revealOrStart = (row, col) => {
    if (gameStatus.value === STATUS.READY) {
        while(true) {
            setupBoard()
            if (board.value[row][col].isMine) {
                continue
            }
            if (board.value[row][col].adjacentMines !== 0) {
                continue
            }
            reveal(row, col)
            break
        }
        gameStatus.value = STATUS.PLAYING
    } else {
        reveal(row, col)
    }
}

const toggleFlag = (row, col) => {
    if (gameStatus.value === STATUS.READY) {
        return
    }

    if (board.value[row][col].isRevealed) {
        return
    }

    board.value[row][col].isFlagged = !board.value[row][col].isFlagged
}

const revealAdj = (row, col) => {
    if (gameStatus.value !== STATUS.PLAYING) {
        return
    }
    
    // if not all flags are placed, return
    let notFlaggedList = []
    for (let i = -1; i <= 1; i++) {
        for (let j = -1; j <= 1; j++) {
            if (i === 0 && j === 0) {
                continue
            }

            let newRow = row + i
            let newCol = col + j

            if (newRow < 0 || newRow >= 20 || newCol < 0 || newCol >= 23) {
                continue
            }

            if (!board.value[newRow][newCol].isFlagged) {
                notFlaggedList.push([newRow, newCol])
            }
        }
    }

    if ((8 - notFlaggedList.length) !== board.value[row][col].adjacentMines) {
        for (let [row, col] of notFlaggedList) {
            if (board.value[row][col].isRevealed) {
                continue
            }
            if (board.value[row][col].isFlagged) {
                continue
            }
            board.value[row][col].highlight = true
        }
        setTimeout(() => {
            for (let [row, col] of notFlaggedList) {
                board.value[row][col].highlight = false
            }
        }, 100);
        return
    }

    for (let i = -1; i <= 1; i++) {
        for (let j = -1; j <= 1; j++) {
            if (i === 0 && j === 0) {
                continue
            }

            let newRow = row + i
            let newCol = col + j

            if (newRow < 0 || newRow >= 20 || newCol < 0 || newCol >= 23) {
                continue
            }

            reveal(newRow, newCol)
        }
    }
}

</script>

<!--
MineSweeper Board.
spec:
- 23 * 20 grid
- 99 mines
- structure:
    (div) -> (div) * 20 -> (div) * 23

- data:
    2d array of objects:
    {
        isMine: boolean,
        isRevealed: boolean,
        value: {
            isFlagged: boolean
            or
            adjacentMines: number
        }
    }
-->

<template>
    <div class="board unselectable" @contextmenu.prevent>
        <div class="row" v-for="row in board">
            <div class="cell" v-for="cell in row" 
            :class="{
                highlight: cell.highlight,
                revealed: cell.isRevealed,
                hidden: !cell.isRevealed,
                flagged: cell.isFlagged,
                lost: gameStatus === STATUS.LOST,
                lostRevealed: cell.isMine && cell.isRevealed,
                lostMine: cell.isMine && gameStatus === STATUS.LOST,
            }"
            @click="revealOrStart(cell.row, cell.col)"
            @mousedown.right="toggleFlag(cell.row, cell.col)"
            @dblclick="revealAdj(cell.row, cell.col)"
            >
                {{ cellAsChar(cell) }}
            </div>
        </div>
    </div>
</template>


<style scoped>
.row {
    display: block;
    margin: 0;
}
.cell {
    width: 25px;
    height: 25px;
    margin: 1px;
    border: 1px solid black;
    display: inline-block;
    border-radius: 2px;
}


.highlight {
    border-color: aqua;
}

.revealed {
    background-color: lightgreen;
}
.hidden {
    color:transparent;
    background-color: lightgray;;
}
.flagged {
    background-color: lightcoral;
}
.lost {
    background-color: gray;
}
.lostRevealed {
    background-color: red;
}
.lostMine {
    color:red;
}

.unselectable {
  user-select: none;
  -moz-user-select: none;
  -webkit-user-select: none;
  -ms-user-select: none;
}

</style>