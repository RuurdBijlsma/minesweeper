<template>
    <div class="mine-sweeper">
        <div class="title">
            <h1 v-if="game.status === ''">Mine Sweeper</h1>
            <h1 v-else>{{game.status}}</h1>
        </div>
        <div class="canvas-container-outer" ref="canvasContainer">
            <div class="canvas-container-inner">
                <canvas
                        ref="canvas"
                        class="canvas"
                        @touchstart="touchStart"
                        @touchmove="touchMove"
                        @touchend="touchEnd"
                        @contextmenu="clickCanvas"
                        @click="clickCanvas"
                ></canvas>
            </div>
        </div>
        <div class="controls">
            <div class="text-fields">
                <v-text-field type="number" label="Width" v-model="grid.width"></v-text-field>
                <v-text-field type="number" label="Height" v-model="grid.height"></v-text-field>
                <v-text-field type="number" label="Bombs" v-model="game.nBombs" :rules="bombRules"></v-text-field>
            </div>
            <v-btn @click="newGame" color="primary" class="new-game-button">New Game</v-btn>
        </div>
    </div>
</template>

<script>
    const initialGridSize = window.innerWidth > 600 ? 20 : 12;
    const initialBombs = initialGridSize === 20 ? 60 : 30;
    export default {
        name: 'MineSweeper',
        data: () => ({
            animation: -1,
            canvasRatio: 1,
            grid: {
                width: initialGridSize,
                height: initialGridSize,
                cells: [],
            },
            game: {
                status: '',
                nBombs: initialBombs,
                actualBombs: 0,
                showBombs: false,
                isDead: false,
            },
            firstClick: true,
            images: {
                bomb: new Image(),
                flag: new Image(),
                cell: new Image(),
                numbers: {
                    1: new Image(),
                    2: new Image(),
                    3: new Image(),
                    4: new Image(),
                    5: new Image(),
                    6: new Image(),
                    7: new Image(),
                    8: new Image(),
                }
            },
            touchTime: false,
            hold: {
                timeout: -1,
                x: 0, y: 0,
                down: true,
            },
            holdDuration: 300,
            preventClick: false,
        }),
        mounted() {
            this.canvas = this.$refs.canvas;
            this.context = this.canvas.getContext('2d');

            this.images.bomb.src = 'img/bomb.png';
            this.images.flag.src = 'img/flag.png';
            this.images.cell.src = 'img/cell.png';
            for (let i = 1; i <= 8; i++)
                this.images.numbers[i].src = `img/${i}.png`;

            this.animation = requestAnimationFrame(this.render);
            window.addEventListener('resize', this.resizeCanvas, false);
            this.newGame();
        },
        beforeDestroy() {
            cancelAnimationFrame(this.animation);
            window.removeEventListener('resize', this.resizeCanvas);
            clearTimeout(this.hold.timeout);
        },
        methods: {
            newGame() {
                this.canvasRatio = this.grid.width / this.grid.height;
                this.resizeCanvas();
                this.firstClick = true;
                this.grid.cells = this.getCells();
                this.game.status = '';
                this.game.showBombs = false;
                this.game.isDead = false;
            },
            getCells() {
                let cells = [];
                for (let y = 0; y < this.grid.height; y++) {
                    for (let x = 0; x < this.grid.width; x++) {
                        cells.push({
                            x, y,
                            revealed: false,
                            bomb: false,
                            flag: false,
                            bombNeighbours: 0,
                        });
                    }
                }
                return cells;
            },
            generateField(startPos) {
                let cells = this.getCells();

                let notBombs = this.getNeighbours(...startPos).map(pos => this.getCell(...pos, cells)).filter(n => n !== undefined);
                notBombs.push(this.getCell(...startPos, cells));
                console.log(notBombs);

                if (this.game.nBombs > cells.length - notBombs.length)
                    console.warn("There are move bombs than grid cells, clamping game.nBombs to", cells.length - notBombs.length);
                let nBombs = Math.min(this.game.nBombs, cells.length - notBombs.length);
                this.game.actualBombs = nBombs;
                notBombs.forEach(cell => cells.splice(cells.indexOf(cell), 1));
                let bombCells = [];

                for (let i = 0; i < nBombs; i++) {
                    let randomIndex = Math.floor(Math.random() * cells.length);
                    let bombCell = cells.splice(randomIndex, 1)[0];
                    bombCell.bomb = true;
                    bombCells.push(bombCell);
                }
                cells = cells.concat(bombCells);
                cells = cells.concat(notBombs);

                for (let cell of cells) {
                    cell.neighbours = this.getNeighbours(cell.x, cell.y)
                        .map(([x, y]) => this.getCell(x, y, cells))
                        .filter(c => c !== undefined);
                    cell.bombNeighbours = cell.neighbours
                        .map(c => c.bomb)
                        .reduce((a, b) => a + b, 0);
                }
                return cells;
            },
            getNeighbours(x, y, neigh8 = true) {
                if (neigh8)
                    return [
                        [x - 1, y + 0],//left
                        [x + 0, y - 1],//top
                        [x + 1, y + 0],//right
                        [x + 0, y + 1],//bottom
                        [x - 1, y - 1],//top left
                        [x + 1, y - 1],//top right
                        [x + 1, y + 1],//bottom right
                        [x - 1, y + 1],//bottom left
                    ]
                return [
                    [x - 1, y + 0],//left
                    [x + 0, y - 1],//top
                    [x + 1, y + 0],//right
                    [x + 0, y + 1],//bottom
                ]
            },
            touchStart(e) {
                let {top, left} = this.canvas.getBoundingClientRect();
                let x = e.touches[0].clientX - left;
                let y = e.touches[0].clientY - top;
                this.hold.down = true;
                this.hold.x = x;
                this.hold.y = y;
                this.hold.timeout = setTimeout(() => {
                    this.clickPos(x, y, true);
                    this.preventClick = true;
                }, this.holdDuration);
            },
            touchMove(e) {
                let {top, left} = this.canvas.getBoundingClientRect();
                let x = e.touches[0].clientX - left;
                let y = e.touches[0].clientY - top;
                let distance = Math.sqrt((this.hold.x - x) ** 2 + (this.hold.y - y) ** 2);
                if (distance > 30) {
                    console.log("Cancelling hold");
                    this.touchEnd();
                }
            },
            touchEnd() {
                clearTimeout(this.hold.timeout);
                setTimeout(() => {
                    this.preventClick = false;
                }, 50);
            },
            clickCanvas(e) {
                e.preventDefault();
                let isRightClick = e.button === 2;
                let {top, left} = this.canvas.getBoundingClientRect();
                let x = e.clientX - left;
                let y = e.clientY - top;
                this.clickPos(x, y, isRightClick);
            },
            clickPos(x, y, rightClick) {
                if (this.preventClick) {
                    this.preventClick = false;
                    return;
                }
                if (this.game.isDead)
                    return;
                let {width, height} = this.canvas.getBoundingClientRect();
                let cellWidth = width / this.grid.width;
                let cellHeight = height / this.grid.height;
                let cellX = Math.floor(x / cellWidth);
                let cellY = Math.floor(y / cellHeight);
                let cell = this.getCell(cellX, cellY);
                if (rightClick)
                    this.flagCell(cell);
                else
                    this.clickCell(cell);
            },
            flagCell(cell) {
                if (cell.revealed)
                    return;
                cell.flag = !cell.flag;
                let flags = this.grid.cells.filter(cell => cell.flag);
                let correctFlags = flags.filter(cell => cell.bomb);
                console.log(correctFlags, flags, this.game.nBombs);
                if (correctFlags.length === +this.game.actualBombs && correctFlags.length === flags.length)
                    this.game.status = 'You win!';
            },
            clickCell(cell) {
                if (cell.flag)
                    return;
                if (this.firstClick) {
                    this.grid.cells = this.generateField([cell.x, cell.y]);
                    cell = this.getCell(cell.x, cell.y);
                    console.log("First click, continuing?");
                    this.firstClick = false;
                }
                if (cell.bomb) {
                    this.game.status = 'You die!';
                    this.game.isDead = true;
                    this.game.showBombs = true;
                } else {
                    cell.revealed = true;
                    if (cell.bombNeighbours === 0) {
                        cell.neighbours
                            .filter(c => !c.revealed)
                            .forEach(this.clickCell);
                    }
                }
            },
            getCell(x, y, cells = this.grid.cells) {
                for (let cell of cells) {
                    if (cell === undefined)
                        console.log(cells, cell);
                }
                return cells.find(cell => cell.x === x && cell.y === y);
            },
            render() {
                this.animation = requestAnimationFrame(this.render);

                this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);

                this.drawBackground();
                this.drawGridLines();
                this.drawCells();
            },
            drawBackground() {
                this.context.fillStyle = '#bdbdbd';
                this.context.fillRect(0, 0, this.canvas.width, this.canvas.height);
            },
            drawGridLines() {
                let cellWidth = 16;
                let cellHeight = 16;

                this.context.strokeStyle = '#7b7b7b';
                this.context.lineWidth = 1;
                this.context.beginPath();
                for (let x = 0; x <= this.grid.width; x++) {
                    this.context.moveTo(0.5 + x * cellWidth, -0.5);
                    this.context.lineTo(0.5 + x * cellWidth, this.canvas.height + 0.5);
                }
                for (let y = 0; y <= this.grid.height; y++) {
                    this.context.moveTo(-0.5, 0.5 + y * cellHeight);
                    this.context.lineTo(this.canvas.width + 0.5, 0.5 + y * cellHeight);
                }
                this.context.stroke();
            },
            drawCells() {
                let cellWidth = 16;
                let cellHeight = 16;

                for (let cell of this.grid.cells) {
                    if (cell.bomb && this.game.showBombs) {
                        this.context.drawImage(this.images.bomb,
                            cell.x * cellWidth,
                            cell.y * cellHeight,
                            cellWidth,
                            cellHeight,
                        );
                    } else if (cell.flag) {
                        this.context.drawImage(this.images.flag,
                            cell.x * cellWidth,
                            cell.y * cellHeight,
                            cellWidth,
                            cellHeight,
                        );
                    } else if (cell.revealed && cell.bombNeighbours > 0) {
                        this.context.drawImage(this.images.numbers[cell.bombNeighbours],
                            cell.x * cellWidth,
                            cell.y * cellHeight,
                            cellWidth,
                            cellHeight,
                        );
                    } else if (cell.revealed) {
                    } else {
                        this.context.drawImage(this.images.cell,
                            cell.x * cellWidth,
                            cell.y * cellHeight,
                            cellWidth,
                            cellHeight,
                        );
                    }
                }
            },
            resizeCanvas() {
                let {width, height} = this.$refs.canvasContainer.getBoundingClientRect();
                let containerRatio = width / height;
                let canvasWidth, canvasHeight;
                if (containerRatio > this.canvasRatio) {
                    //Container is wider than canvas is supposed to be
                    // noinspection JSSuspiciousNameCombination
                    canvasWidth = height * this.canvasRatio;
                    canvasHeight = height;
                } else {
                    //Container is taller than canvas is supposed to be
                    canvasWidth = width;
                    // noinspection JSSuspiciousNameCombination
                    canvasHeight = width / this.canvasRatio;
                }
                this.canvas.style.width = canvasWidth + 'px';
                this.canvas.style.height = canvasHeight + 'px';
                this.canvas.width = this.grid.width * 16 + 1;
                this.canvas.height = this.grid.height * 16 + 1;
            },
        },
        watch: {
            'grid.width'() {
                this.newGame();
            },
            'grid.height'() {
                this.newGame();
            },
            'game.nBombs'() {
                this.newGame();
            },
        },
        computed: {
            maxBombs() {
                let maxBombs = this.grid.width * this.grid.height;
                let clearSpaces = Math.min(this.grid.width, this.grid.height, 3) * 3;
                console.log(maxBombs - clearSpaces)
                return maxBombs - clearSpaces;
            },
            bombRules() {
                return [
                    v => !!v || 'Required',
                    v => v >= 1 || 'Number of bombs must be > 0',
                    v => v <= this.maxBombs || 'Number of bombs can at most be ' + this.maxBombs,
                ]
            }
        }
    }
</script>
<style scoped>
    .mine-sweeper {
        background-color: rgb(40, 40, 40);
        flex: 1;
        display: flex;
        flex-direction: column;
        text-align: center;
    }

    .title {
        height: auto;
        padding: 20px;
        display: inline-block;
    }

    @media (max-width: 424px) {
        .title {
            height: 136px;
        }
    }

    .canvas-container-outer {
        width: 100%;
        flex-grow: 1;
        display: flex;
        justify-content: center;
        flex-direction: column;
    }

    .canvas-container-inner {
        width: 100%;
        display: flex;
        justify-content: center;
        flex-direction: row;
    }

    .canvas {
        background-color: grey;
        image-rendering: -moz-crisp-edges;
        image-rendering: -webkit-crisp-edges;
        image-rendering: pixelated;
        image-rendering: crisp-edges;
    }

    .controls {
        display: flex;
        flex-direction: column;
        max-width: 700px;
        width: calc(100% - 30px);
        margin: 20px auto;
        padding: 0 15px;
    }

    .text-fields {
        display: flex;
        justify-content: space-around;
    }

    .text-fields > * {
        margin: 0 5px;
    }
</style>