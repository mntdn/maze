<script setup lang="ts">
import { ref } from 'vue'

export interface DirectionResult {
    line: number,
    col: number,
    isOK: boolean,
}

const TOP = 1;
const RIGHT = 2;
const BOTTOM = 4;
const LEFT = 8;
const VISITED = 16;

const mazeSize = ref(35);
const mazeGrid = ref([[]] as number[][])

const getClass = (col: number) => {
    let result: string[] = [];
    if (col & TOP) result.push("top");
    if (col & RIGHT) result.push("right");
    if (col & BOTTOM) result.push("bottom");
    if (col & LEFT) result.push("left");
    if (col & VISITED) result.push("visited");
    return result;
}

const randomInt = (_min: number, _max: number) => {
    let min = Math.ceil(_min);
    let max = Math.floor(_max);
    return Math.floor(Math.random() * (max - min + 1)) + min;
}

const getRandomRoom = () => {
    return randomInt(1, 15);
}

const getRandomDirection = (forbidden: number[]) => {
    let result = 1 << randomInt(0, 3);
    while (forbidden.indexOf(result) >= 0)
        result = 1 << randomInt(0, 3);
    return result;
}

const getOpposite = (direction: number): number => {
    return direction > 2 ? direction >> 2 : direction << 2;
}

// génère un ensemble de pièces complètement aléatoire, avec tous les côtés fermés
const randMaze = () => {
    mazeGrid.value = [];
    for (let i = 0; i < mazeSize.value; i++) {
        let line = [];
        for (let j = 0; j < mazeSize.value; j++) {
            let r = getRandomRoom();
            if (i === 0) r |= TOP;
            if (j === 0) r |= LEFT;
            if (i === mazeSize.value - 1) r |= BOTTOM;
            if (j === mazeSize.value - 1) r |= RIGHT;
            line.push(r);
        }
        mazeGrid.value.push(line);
    }
}

// casse un mur au hasard (mais sans quitter le labyrinthe) et renvoie la direction
// dans lequel on a cassé
const breakRandomWall = (line: number, col: number): number => {
    let forbidden = [];
    if (line === 0) forbidden.push(TOP);
    if (col === 0) forbidden.push(LEFT);
    if (line === mazeSize.value - 1) forbidden.push(BOTTOM);
    if (col === mazeSize.value - 1) forbidden.push(RIGHT);
    let r = getRandomDirection(forbidden);
    switch (r) {
        case TOP:
            mazeGrid.value[line][col] &= ~TOP;
            mazeGrid.value[line - 1][col] &= ~BOTTOM;
            break;
        case RIGHT:
            mazeGrid.value[line][col] &= ~RIGHT;
            mazeGrid.value[line][col + 1] &= ~LEFT;
            break;
        case BOTTOM:
            mazeGrid.value[line][col] &= ~BOTTOM;
            mazeGrid.value[line + 1][col] &= ~TOP;
            break;
        case LEFT:
            mazeGrid.value[line][col] &= ~LEFT;
            mazeGrid.value[line][col - 1] &= ~RIGHT;
            break;
        default:
            break;
    }
    return r;
}

// "double" tous les murs, sauf pour les côtés
// puis évite les rooms sans sortie en "creusant" un côté au hasard
const reinforce = () => {
    for (let i = 0; i < mazeSize.value; i++) {
        for (let j = 0; j < mazeSize.value; j++) {
            if (j < mazeSize.value - 1) {
                if ((mazeGrid.value[i][j] & RIGHT) || (mazeGrid.value[i][j + 1] & LEFT)) {
                    mazeGrid.value[i][j] |= RIGHT; mazeGrid.value[i][j + 1] |= LEFT;
                }
            }
            if (i < mazeSize.value - 1) {
                if ((mazeGrid.value[i][j] & BOTTOM) || (mazeGrid.value[i + 1][j] & TOP)) {
                    mazeGrid.value[i][j] |= BOTTOM; mazeGrid.value[i + 1][j] |= TOP;
                }
            }
            if (mazeGrid.value[i][j] === 15) {
                // si on a une pièce sans entrée ou sortie, on creuse un mur au hasard
                breakRandomWall(i, j);
            }
        }
    }
}

// renvoie la nouvelle coordonnée
const addDir = (line: number, col: number, direction: number): DirectionResult => {
    let isOK = false;
    let l = line, c = col;
    if (!(
        (direction & mazeGrid.value[line][col]) ||
        (direction === TOP && line === 0) || (direction === RIGHT && col === mazeSize.value - 1) ||
        (direction === BOTTOM && line === mazeSize.value - 1) || (direction === LEFT && col === 0))) {

        isOK = true;
        if (direction === TOP) l--;
        if (direction === RIGHT) c++;
        if (direction === BOTTOM) l++;
        if (direction === LEFT) c--;
    }
    return { line: l, col: c, isOK: isOK };
}

const walkAndCrashRec = (line: number, col: number, direction: number, forbidden: number[], i: number) => {
    // on marque la pièce visitée
    mazeGrid.value[line][col] |= VISITED;
    let res = addDir(line, col, direction);
    if (i === 10)
        return;
    if (res.isOK) {
        walkAndCrashRec(res.line, res.col, direction, [...forbidden, getOpposite(direction)], ++i);
        console.log("YES", res.line, res.col, direction, [getOpposite(direction)], i);
    }
    else {
        let s = new Set([...forbidden, direction]);
        if (s.size === 4)
            return;
        let newDir = getRandomDirection([...forbidden, direction]);
        console.log("NO", line, col, newDir, [...forbidden, direction, newDir], i);
        walkAndCrashRec(line, col, newDir, [...forbidden, direction, newDir], ++i);
    }
}

// l'idée est de casser un mur à chaque fois qu'on est bloqué un certain nb de fois
const walkAndCrash = (line: number, col: number, direction: number) => {
    // on marque la pièce visitée
    let forbidden: number[] = [getOpposite(direction)];
    let finished = false;
    let curDir = direction;
    let l = line, c = col;
    let nbOfCrashes = 0;
    while (nbOfCrashes < 10) {
        mazeGrid.value[l][c] |= VISITED;
        let res = addDir(l, c, curDir);
        if (res.isOK) {
            l = res.line; c = res.col; forbidden = [...forbidden, getOpposite(curDir)];
            // console.log("YES", l, c);
        }
        else {
            let foundExit = false;
            forbidden = [...forbidden, curDir];
            do {
                let newDir = getRandomDirection(forbidden);
                let newRoom = addDir(l, c, newDir);
                // console.log("NO", l, c, forbidden, newDir, newRoom);
                if (newRoom.isOK) {
                    l = newRoom.line; c = newRoom.col; curDir = newDir; forbidden = [getOpposite(newDir)];
                    break;
                } else {
                    forbidden = [...forbidden, newDir];
                }
                forbidden = [...new Set([...forbidden])];
            } while (forbidden.length < 4)
            if (forbidden.length >= 4) {
                nbOfCrashes++;
                curDir = breakRandomWall(l, c);
                forbidden = [getOpposite(curDir)];
                // console.log("newDir", curDir);
            }
        }
    }
}

const chooseOpening = () => {
    let r = getRandomDirection([]);
    let pos = randomInt(0, mazeSize.value - 1);
    let line = 0, col = 0;
    if (r === TOP) col = pos;
    if (r === RIGHT) { line = pos; col = mazeSize.value - 1; }
    if (r === BOTTOM) { line = mazeSize.value - 1; col = pos; }
    if (r === LEFT) line = pos;
    mazeGrid.value[line][col] &= ~r;

    // on va dans la direction opposée
    let dir = getOpposite(r);
    console.log(line, col, dir, [r], 0);
    walkAndCrash(line, col, dir);
}
randMaze();
</script>

<template>
    <div class="maze" :style="{width: mazeSize * 20 + 'px'}">
        <template v-for="(line, i) in mazeGrid" :key="i">
            <template v-for="(col, j) in line" :key="i*j">
                <div class="room" :class="getClass(col)"></div>
            </template>
        </template>
    </div>
    <button @click="randMaze()">N</button>
    <button @click="reinforce()">C</button>
    <button @click="chooseOpening()">O</button>
</template>

<style scoped lang="scss">
.maze {
    line-height: 0px;

    .room {
        width: 20px;
        height: 20px;
        display: inline-block;
        box-sizing: border-box;

        &.top {
            border-top: 1px solid black;
        }

        &.right {
            border-right: 1px solid black;
        }

        &.bottom {
            border-bottom: 1px solid black;
        }

        &.left {
            border-left: 1px solid black;
        }

        &.visited {
            background-color: lightblue;
        }

        &.top.right.bottom.left {
            background-color: red;
        }
    }
}
</style>
