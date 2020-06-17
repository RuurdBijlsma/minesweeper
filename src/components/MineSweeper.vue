<template>
    <div class="mine-sweeper">
        <h1>Mine Sweeper {{status}}</h1>
        <div class="canvas-container-outer" ref="canvasContainer">
            <div class="canvas-container-inner">
                <canvas
                        ref="canvas"
                        class="canvas"
                        @contextmenu="clickCanvas"
                        @click="clickCanvas"
                ></canvas>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'MineSweeper',
        data: () => ({
            animation: -1,
            canvasRatio: 1,
            grid: {
                width: 20,
                height: 20,
                cells: [],
            },
            game: {
                nBombs: 50,
            },
            status: '',
            showBombs: false,
        }),
        mounted() {
            this.canvas = this.$refs.canvas;
            this.context = this.canvas.getContext('2d');

            this.animation = requestAnimationFrame(this.render);
            window.addEventListener('resize', this.resizeCanvas, false);

            this.resizeCanvas();
            this.grid.cells = this.generateField();
            console.log(this.grid.cells);
        },
        beforeDestroy() {
            cancelAnimationFrame(this.animation);
            window.removeEventListener('resize', this.resizeCanvas);
        },
        methods: {
            generateField() {
                let cells = [];
                for (let y = 0; y <= this.grid.height; y++) {
                    for (let x = 0; x <= this.grid.width; x++) {
                        cells.push({
                            x, y,
                            show: 'cell',
                            bomb: false,
                            flag: false,
                            bombNeighbours: 0,
                        });
                    }
                }
                if (this.game.nBombs > cells.length)
                    console.warn("There are move bombs than grid cells, clamping nBombs to", cells.length);
                let nBombs = Math.min(this.game.nBombs, cells.length);
                let bombCells = [];
                for (let i = 0; i < nBombs; i++) {
                    let randomIndex = Math.floor(Math.random() * cells.length);
                    let bombCell = cells.splice(randomIndex, 1)[0];
                    bombCell.bomb = true;
                    bombCells.push(bombCell);
                }
                cells = cells.concat(bombCells);

                for (let cell of cells) {
                    cell.neighbours = this.getNeighbours(cell.x, cell.y)
                        .map(([x, y]) => this.getCell(x, y, cells))
                        .filter(c => c !== undefined);
                    cell.bombNeighbours = cell.neighbours
                        .map(c => c.bomb)
                        .reduce((a, b) => a + b);
                }

                return cells;
            },
            getNeighbours(x, y, neigh8 = true) {
                if (neigh8)
                    return [
                        [x - 1, y + 0],//left
                        [x - 1, y - 1],//top left
                        [x + 0, y - 1],//top
                        [x + 1, y - 1],//top right
                        [x + 1, y + 0],//right
                        [x + 1, y + 1],//bottom right
                        [x + 0, y + 1],//bottom
                        [x - 1, y + 1],//bottom left
                    ]
                return [
                    [x - 1, y + 0],//left
                    [x + 0, y - 1],//top
                    [x + 1, y + 0],//right
                    [x + 0, y + 1],//bottom
                ]
            },
            clickCanvas(e) {
                e.preventDefault();
                let {top, left} = this.canvas.getBoundingClientRect();
                let x = e.pageX - left;
                let y = e.pageY - top;
                let cellWidth = this.canvas.width / this.grid.width;
                let cellHeight = this.canvas.height / this.grid.height;
                let cellX = Math.floor(x / cellWidth);
                let cellY = Math.floor(y / cellHeight);
                let cell = this.getCell(cellX, cellY);
                if (e.button === 2)
                    this.flagCell(cell);
                else
                    this.clickCell(cell);
            },
            flagCell(cell) {
                cell.flag = !cell.flag;
                if (cell.flag)
                    cell.show = 'flag';
                else
                    cell.show = 'cell';
                let flaggedCells = this.grid.cells.filter(cell => cell.flag)
                console.log(flaggedCells)
                if (flaggedCells.length === this.game.nBombs) {
                    this.status = 'You win!';
                }
            },
            clickCell(cell) {
                if (cell.bomb) {
                    this.status = 'You die!';
                    this.showBombs = true;
                } else {
                    if (cell.bombNeighbours > 0) {
                        cell.show = 'number';
                    } else {
                        cell.show = 'empty';
                        cell.neighbours
                            .filter(c => c.show !== 'empty')
                            .forEach(this.clickCell);
                    }
                }
            },
            getCell(x, y, cells = this.grid.cells) {
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
                this.context.fillStyle = 'black';
                this.context.fillRect(0, 0, this.canvas.width, this.canvas.height);
            },
            drawGridLines() {
                let cellWidth = this.canvas.width / this.grid.width;
                let cellHeight = this.canvas.height / this.grid.height;

                this.context.strokeStyle = 'red';
                this.context.lineWidth = this.canvas.width / 400;
                this.context.beginPath();
                for (let x = 0; x <= this.grid.width; x++) {
                    this.context.moveTo(x * cellWidth, 0);
                    this.context.lineTo(x * cellWidth, this.canvas.height);
                }
                for (let y = 0; y <= this.grid.height; y++) {
                    this.context.moveTo(0, y * cellHeight);
                    this.context.lineTo(this.canvas.width, y * cellHeight);
                }
                this.context.stroke();
            },
            drawCells() {
                // console.log(this.grid.cells);
                let cellWidth = this.canvas.width / this.grid.width;
                let cellHeight = this.canvas.height / this.grid.height;

                let padding = this.context.lineWidth;
                for (let cell of this.grid.cells) {
                    switch (cell.show) {
                        case 'flag':
                            this.context.fillStyle = 'red';
                            this.context.fillRect(
                                cell.x * cellWidth + padding,
                                cell.y * cellHeight + padding,
                                cellWidth - padding * 2,
                                cellHeight - padding * 2
                            );
                            break;
                        case 'empty':
                            this.context.fillStyle = 'gray';
                            this.context.fillRect(
                                cell.x * cellWidth + padding,
                                cell.y * cellHeight + padding,
                                cellWidth - padding * 2,
                                cellHeight - padding * 2
                            );
                            break;
                        case 'cell':
                            if (cell.bomb && this.showBombs) {
                                this.context.fillStyle = 'blue';
                                this.context.fillRect(
                                    cell.x * cellWidth + padding,
                                    cell.y * cellHeight + padding,
                                    cellWidth - padding * 2,
                                    cellHeight - padding * 2
                                );
                            }
                            break;
                        case 'number':
                            this.context.fillStyle = 'white';
                            let textHeight = cellHeight - padding * 2;
                            this.context.font = `${textHeight}px Roboto`;
                            let textWidth = this.context.measureText(cell.bombNeighbours).width;
                            this.context.fillText(
                                cell.bombNeighbours,
                                cell.x * cellWidth + cellWidth / 2 - textWidth / 2,
                                cell.y * cellHeight + cellHeight / 2 + textHeight / 2.5,
                            )
                            break;
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
                    canvasWidth = height;
                    canvasHeight = height;
                } else {
                    //Container is taller than canvas is supposed to be
                    canvasWidth = width;
                    // noinspection JSSuspiciousNameCombination
                    canvasHeight = width;
                }
                this.canvas.style.width = canvasWidth + 'px';
                this.canvas.style.height = canvasHeight + 'px';
                this.canvas.width = canvasWidth;
                this.canvas.height = canvasHeight;
            },
        },
    }
</script>
<style scoped>
    .mine-sweeper {
        background-color: rgb(40, 40, 40);
        height: 100%;
        width: 100%;
        display: flex;
        flex-direction: column;
        text-align: center;
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
    }
</style>