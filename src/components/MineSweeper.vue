<template>
    <div class="game-container">
         <div class="panel-container">
            <div class="panel-data-container">
                <span
                    class="iconfont"
                    style="font-size: 60px;"
                >&#xe778;</span>
                
                <div>
                    {{ selectedMineCount }} / {{ mineCount }}
                </div>
            </div>
            <div>
                <button
                    class="mine-sweeper-button"
                    @click="reStart"
                >
                    重开一局
                </button>

                <button
                    class="mine-sweeper-button"
                    style="margin-top: 15px;"
                    @click="$emit('selectDifficulty')"
                >
                    改变难度
                </button>
            </div>
        </div>


        <div
            class="mine-sweeper-container"
            @contextmenu.prevent
        >
            <div
                v-for="i in height"
                :key="i"
                class="mine-sweeper-row"
            >
                <div
                    v-for="j in width"
                    :key="j"
                    class="mine-sweeper-item"
                    :class="{'is-open':openStatus[(i - 1) * width + j - 1]}"
                    
                    @click.right="handleRightClick(i-1,j-1)"
                    @click.left="handleLeftClick (i-1, j-1)"
                    @click="start()"
                >
                    <span
                        v-if="markStatus[(i-1)*width+(j-1)] == 1"
                        class="iconfont" 
                    >&#xe778;</span>
                    
                    <span
                        v-else-if="markStatus[(i-1)*width+(j-1)] == 2"
                        class="iconfont"
                    >&#xe720;</span>
                    <template v-else-if="openStatus[(i-1)*width+(j-1)]">
                        <span
                            v-if="mines[((i-1)*width+j-1)]"
                            class="iconfont"
                        >&#xe63a;</span>
                        <span v-if="neighbourMineCount[(i-1)*width+(j-1)]>0">
                            {{ neighbourMineCount[(i-1)*width+(j-1)] }}
                        </span>
                        
                    </template>
                </div>
            </div>
        </div>
        
    </div>
</template>

<script>
function shuffle (mines, start) {//把炸弹打乱洗牌
    for (let i = start; i < mines.length; i++) {
        const randomIndex = Math.floor(Math.random() * (i + 1));
        const tmp = mines[randomIndex];
        mines[randomIndex] = mines[i];
        mines[i] = tmp;
    }
}

export default {
    props: {
        play: {
            type: Boolean,
            required: true,
        },
        width: {
            type: Number,
            required: true,
        },
        height: {
            type: Number,
            required: true,
        },
        mineCount: {
            type: Number,
            required: true,
        },
    },
    data () {
        return {
            isEnd: false,
            mines: [],//炸弹组
            openStatus: [],//已经被点击
            markStatus: [],//已经标记
            selectedMineCount: 0,
            loop: 0
        };
    },
    computed: {
        neighbourMineCount () {////找出炸弹，并对他的邻居进行标记
            const result = new Array(this.width * this.height).fill(0);
            for (let i = 0; i < result.length; i++) {
                if (!this.mines[i]) {
                    continue;
                }
                const y = i % this.width;
                const x = (i - y) / this.width;
                for (let j = -1; j < 2; j++) {
                    const newX = x + j;
                    if (newX < 0 || newX === this.height) {
                        continue;
                    }
                    for (let k = -1; k < 2; k++) {
                        const newY = y + k;
                        if (newY < 0 || newY === this.width) {
                            continue;
                        }
                        result[newX * this.width + newY]++;
                    }
                }
            }
            return result;
        },
    },
    watch: {
        play () {
            if (!this.play) {
                return;
            }
            this.init(this.width, this.height, this.mineCount);
        },
        selectedMineCount () {//判断是否赢得比赛
            if (this.selectedMineCount === this.mineCount) {
                const match = this.mines.every((isMine, index) => {
                    if ((isMine && this.markStatus[index] === 1) || (!isMine && this.markStatus[index] !== 1)) {
                        return true;
                    }
                    return false;
                });
                if (match) {
                    this.$nextTick(() => {
                        alert('win');
                    });
                    this.isEnd = true;
                }
            }
        },
    },
    methods: {
        reStart () {
            this.init(this.width, this.height, this.mineCount);
        },
        init (width, height, mineCount) {
            this.isEnd = false;
            const total = width * height;
            const mines = new Array(total).fill(0);
            for (let i = 0; i < mineCount; i++) {
                mines[i] = 1;//各个位置炸弹存在的情况
            }
            shuffle(mines, mineCount);
            this.mines = mines;
            this.openStatus = new Array(total).fill(0);
            this.markStatus = new Array(total).fill(0);
            this.selectedMineCount = 0;
        },
        handleLeftClick (x, y) {
            const index = x * this.width + y;
            console.log('周围有'+this.neighbourMineCount[index]+'个雷');
            if (this.isEnd) {
                return;
            }
            if (this.openStatus[index] === 1 || this.markStatus[index] === 1) {
                //如果这个格子已经点出来了或者标记了，返回
                return;
            }

            if (this.mines[index]) {//刚好点到炸弹
               this.openStatus.splice(index, 1, 1);//显示炸弹动画
               // this.openStatus[index]===3;
               console.log('你中雷了');
                this.$nextTick(() => {
                    alert('mine');
                });
                this.isEnd = true;
                return;
            }
            if (this.neighbourMineCount[index] > 0) {//刚好点到非空邻居节点上
                this.openStatus.splice(index, 1, 1);
                return;
            }
            //若点到为零的邻居点，即空白点，执行floodfill函数
            this.floodfill(x, y);
        },
        ale(){
            alert('mine');
        },
        floodfill (x, y) {//若此点周围邻居都是0，向潮水一样扩散，直到碰到边界或者大于零的邻居节点
            if (x < 0 || y < 0 || x === this.height || y === this.width) {
                return;
            }
            const index = x * this.width + y;
            if (this.openStatus[index] === 1) {
                return;
            }
            this.openStatus.splice(index, 1, 1);
            if (this.neighbourMineCount[index] > 0) {
                return;
            }
            for (let i = -1; i < 2; i++) {
                for (let j = -1; j < 2; j++) {
                    this.floodfill(x + i, y + j);
                }
            }
        },
        handleRightClick (x, y) {
            console.log('标记('+x+','+y+')点');
            if (this.isEnd) {
                return;
            }
            const index = x * this.width + y;
            if (this.openStatus[index] === 1) {
                return;
            }
            this.markStatus.splice(index, 1, (this.markStatus[index] + 1) % 3);
            if (this.markStatus[index] === 1) {
                this.selectedMineCount++;
            } else if (this.markStatus[index] === 2) {
                this.selectedMineCount--;
            }
        },
        start (){

            clearTimeout(this.loop);//再次清空定时器，防止重复注册定时器

            this.loop=setTimeout(() => {

           // console.log("长按了");

        },  1000);

    },

 end(){

    clearTimeout(this.loop); //清空定时器，防止重复注册定时器

    },
    }
};
</script>

<style scoped>
.game-container {
    display: flex;
    justify-content: stretch;
    padding: 0 15px;
}

.mine-sweeper-container {
    overflow: hidden;
    width: fit-content;
    margin: 0 auto;
    background-color: #f2f1f0;
}

.mine-sweeper-row {
    display: flex;
}

.mine-sweeper-item {
    width: 40px;
    height: 40px;
    margin: 2px;
    /* line-height: 50px; */
    font-size: 34px;
    text-align: center;
    background-color: #babdb6;
    cursor: pointer;
}

.mine-sweeper-item.is-open {
    background-color: #dededc;
}

.panel-container {
    width: 180px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
}

.panel-data-container {
    padding-top: 15px;
    text-align: top;
}

</style>
