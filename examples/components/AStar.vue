<template>
  <div>
    <h6>点击第一下空白方块设定起点(起点为红色方块) <div class="column red"></div>，点击第二个空白方块设置终点（再次点击重置起点）。</h6>
    <h6>正确的路线起点为红边绿块 <div class="column red green"></div>，终点是蓝边绿块 <div class="column blue green"></div></h6>
    <div class="container m-4">
      <div class="row" v-for="items in elements">
          <div class="column"
               :class="matchClass(item)"
               :id="item.x + '-' + item.y"
               v-for="item in items" @click="setElement(item)"></div>
      </div>
    </div>
    <button type="button" class="button" @click="setRandomObstacle()">随机障碍</button>
    <button type="button" class="button" @click="clearObstacle()">清空障碍</button>
    <button type="button" class="button" @click="setObstacleMode()">障碍模式</button>
    <button type="button" class="button" @click="setStartAndEnd()">起始模式</button>
    <h6>起始模式（默认）：可设置起始点和终点 </h6>
    <h6>障碍模式：点击开启后，可点击方块设置障碍物 </h6>
  </div>
</template>

<script>
export default {
  data() {
    return {
      openList: [],
      closeList: [],
      lineLength: 10,
      elements: [],
      start: null,
      end: null,
      calculating: false,
      range: {
        x: 8,
        y: 8
      },
      obstacles: [],
      mode: 'start-end',
      viablePath: [],
    }
  },
  mounted() {
    this.elements = this.createElements(this.range.y, this.range.x)
  },
  methods: {
    // 计算 G 值
    calculateG(element, length) {
      if (element.previousElement == null) {
        return length
      } else {
        return element.previousElement.g + length
      }
    },
    // 计算 H 值
    calculateH(element) {
      return (Math.abs(element.x - this.end.x) + Math.abs(element.y - this.end.y)) * 10
    },
    // 计算 F 值
    calculateF(element) {
      return element.g + element.h
    },
    // 创建元素阵
    createElements(row, column) {
      let elements = []
      for (let i = 0; i < row; i++) {
        for (let j = 0; j < column; j++) {
          elements[i] || (elements[i] = [])
          let element = {
            status: 1,
            previousElement: null,
            x: 0,
            y: 0,
            f: 0,
            h: 0,
            g: 0,
          }

          element.x = j
          element.y = i

          elements[i][j] = element
        }
      }

      return elements
    },
    // 获取路径
    getPath(element, target) {
      element.previousElement = null

      this.openList.push(element)

      this.search(target)
    },
    // 搜索路径
    search(target) {
      let element = null
      let find = false
      let i = 0
      while(this.openList.length > 0) {
        // 获取 F 最小的元素
        element = this.openList[0]

        // 元素与目标元素相等，则退出循环
        if (element.x == target.x && element.y == target.y) {
          find = true
          break;
        }

        // 上方元素
        if ((element.y - 1) >= 0) {
          this.checkElement(element.x, element.y - 1, element, this.lineLength)
        }

        // 下方元素
        if ((element.y + 1) < this.range.y) {
          this.checkElement(element.x, element.y + 1, element, this.lineLength)
        }

        // 左方元素
        if ((element.x - 1) >= 0) {
          this.checkElement(element.x - 1, element.y, element, this.lineLength)
        }

        // 右方元素
        if ((element.x + 1) < this.range.x) {
          this.checkElement(element.x + 1, element.y, element, this.lineLength)
        }

        // 将 F 值最小的存放到 close list 中
        this.closeList.push(this.openList.shift())

        this.openList.sort(function (prev, next) {
          return prev.f - next.f
        })
      }

      if (find) {
        // 通过终点方格回路获取正确的路径
        this.checkPath(element)
        // 设置绿道
        this.setGreenPath(this.viablePath)
      } else {
        alert('不能寻找正确的路径');
      }
    },
    // 检查并获取正确路径
    checkPath(element) {
      if (element.previousElement != null) {
        this.checkPath(element.previousElement)
      }
      this.viablePath.push(element)
    },
    // 检查元素是否可行
    checkElement(x, y, element, length) {
      let item = this.elements[y][x];
      if (item && item.status == 0) return false;
      // 检查 Close List 列表中是否已存在该元素
      if(this.checkList(this.closeList, item) != -1){
        return false;
      }
      // 检查 Open List 列表中是否已存在该元素
      let index = this.checkList(this.openList, item);
      if(index != -1){
        // 检查当前元素的 G 与父元素的 G 大小比较
        if((item.previousElement.g + length) < this.openList[index].g) {
          this.openList[index].previousElement = element;
          this.openList[index].g = this.calculateG(item, length);
          this.openList[index].f = this.calculateF(item);
        }
      } else {
        this.elements[y][x].g = this.calculateG(item, length);
        this.elements[y][x].h = this.calculateH(item);
        this.elements[y][x].f = this.calculateF(item);
        this.elements[y][x].previousElement = element || null;
        this.openList.push(item);
      }
    },
    // 检查列表中是否已存在该元素
    checkList(list, element) {
      for (var i = 0; i < list.length; i++) {
        if (list[i].x == element.x && list[i].y == element.y) {
          return i
        }
      }
      return -1
    },
    // 设置起点和终点位置
    setElement(item) {
      if (this.mode == 'start-end') {
        if (item.status == 0) return;
        if (this.start && this.end) {
          this.removeGreenPath()
          this.resetStartAndEnd()
          this.closeList = []
          this.openList = []
          this.viablePath = []
          this.elements = this.createElements(this.range.y, this.range.x)
          this.clearElmentPreviousElement()
          this.setObstacle()
        }
        if (this.start == null) {
          this.viablePath = []
          this.start = item
          this.elements[item.y][item.x].status = 2
        } else if (this.end == null) {
          this.end = item
          this.elements[item.y][item.x].status = 3
          this.calculating = true
          this.getPath(this.start, this.end)
        }
      } else if (this.mode == 'obstacle') {
        this.obstacles.push(item)
        this.elements[item.y][item.x].status = 0
      }
    },
    // 随机设置障碍物
    setRandomObstacle() {
      this.clearObstacle()
      for (var i = 0; i < this.elements.length; i++) {
        for (var j = 0; j < this.elements[i].length; j++) {
          if (this.elements[i][j].status == 0) {
            this.elements[i][j].status = 1
          }

          if (Math.floor(Math.random() * 10) < 2) {
            this.elements[i][j].status = 0
            this.obstacles.push(this.elements[i][j])
          }
        }
      }
    },
    // 设置绿色路径
    setGreenPath(list) {
      setTimeout(function () {
        for (let i = 0; i < list.length; i++) {
          document.getElementById(list[i].x + '-' + list[i].y).className += ' green';
        }
      }, 1);
    },
    // 移除绿色路径
    removeGreenPath() {
      let list = this.viablePath;
      for (let i = 0; i < this.viablePath.length; i++) {
        let element = document.getElementById(list[i].x + '-' + list[i].y);
        element.className = element.className.replace(/green/g, '');
      }
      this.viablePath = []
    },
    // 匹配对应的 Class
    matchClass(item) {
      return {
        black: item.status == 0 ? true : false,
        red: item.status == 2 ? true : false,
        blue: item.status == 3 ? true : false
      };
    },
    // 设置为障碍物模式
    setObstacleMode() {
      this.mode = 'obstacle'
    },
    // 设置为起始点设置模式
    setStartAndEnd() {
      this.mode = 'start-end'
    },
    // 设置障碍物
    setObstacle() {
      if (this.obstacles.length > 0) {
        for (var i = 0; i < this.obstacles.length; i++) {
          this.elements[this.obstacles[i].y][this.obstacles[i].x].status = 0
        }
      }
    },
    // 清理障碍物
    clearObstacle() {
      this.obstacles = [];
      this.removeGreenPath();
      this.resetStartAndEnd()
      this.openList = [];
      this.closeList = [];
      this.viablePath = [];
      this.elements = this.createElements(this.range.y, this.range.x)
    },
    // 清理父级方块
    clearElmentPreviousElement() {
      for (var i = 0; i < this.elements; i++) {
        for (var j = 0; i < this.elements[i]; j++) {
          this.elements[i][j].previousElement = null
        }
      }
    },
    // 重置起始点
    resetStartAndEnd() {
      if (this.start) {
        this.elements[this.start.y][this.start.x].status = 1
      }
      if (this.end) {
        this.elements[this.end.y][this.end.x].status = 1
      }
      this.start = null
      this.end = null
    },
  },
}
</script>
