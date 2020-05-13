<template>
  <div class="wraper" ref="wraper">
    <div class="canvas-wraper">
      <canvas id="canvas" ref="canvas"></canvas>
    </div>
    <div class="controlPanel">
      <div :class="[initIdx==idx ? 'contro-item active' : 'contro-item']" v-for="(item,idx) in toolsArr" :key="idx" @click="handleTools(item, idx)">
        <i :class="'iconfont' + item.icon"></i>
      </div>
    </div>
    <div class="download">
      <button type="button" :disabled="done" @click="downLoadImage">转换为base64并预览</button>
      <img :src="imageBase64" v-show="imageBase64!=''" alt="">
    </div>
    </div>
</template>
<script>
import { fabric } from 'fabric'
export default {
  data() {
    return {
      currentTool: 'pick',
      done: false,
      fabricObj: null,
      initIdx: 0,
      toolsArr: [{
          name: 'pick',
          icon: ' icon-move'
        },
        {
          name: 'pencil',
          icon: ' icon-pencil'
        },
        {
          name: 'line',
          icon: ' icon-line'
        },
        {
          name: 'arrow',
          icon: ' icon-arrow'
        },
        {
          name: 'xuxian',
          icon: ' icon-xuxian'
        },
        {
          name: 'text',
          icon: ' icon-ziti'
        },
        {
          name: 'juxing',
          icon: ' icon-juxing'
        },
        {
          name: 'cricle',
          icon: ' icon-yuanxing'
        },
        {
          name: 'ellipse',
          icon: ' icon-tuoyuanxing'
        },
        {
          name: 'equilateral', //三角形
          icon: ' icon-sanjiaoxing'
        },
        {
          name: 'remove',
          icon: ' icon-remove'
        },
        {
          name: 'undo',
          icon: ' icon-huitui'
        },
        {
          name: 'redo',
          icon: ' icon-xiangqian'
        },
        {
          name: 'reset',
          icon: ' icon-reset'
        },
      ],
      mouseFrom: {},
      mouseTo: {},
      lastzoomPoint: {},
      moveCount: 1,
      doDrawing: false,
      isDelete: false,
      fabricHistoryJson: [],
      mods: 0,
      drawingObject: null, //绘制对象
      drawColor: '#E34F51',
      drawWidth: 2,
      imageBase64: '',
      zoom: window.zoom ? window.zoom : 1,
      zoomPoint:new fabric.Point(0,0),//初始时缩放原点的位置设为（0,0），这是页面的左上顶点
      lastzoomPoint:{x:0,y:0},//初始时，前一次缩放原点同样为(0,0)
      lastmousePoint:{x:0,y:0},//进行缩放，需要对此刻缩放位置进行保存，来计算出缩放原点，此刻初始时设为0,0
      lastzoom:1,//表示为上一次的缩放倍数，此刻设为1
      relativeMouseX:0,//表示相对的鼠标位移，用来记录画布的绝对移动距离
      relativeMouseY:0,//表示相对的鼠标位移，用来记录画布的绝对移动距离
    }
  },
  mounted() {
    //初始化canvas 
    this.initCanvas()
  },
  computed: {
    canvasWidth() {
      return window.innerWidth
    }
  },
  methods: {
    initCanvas() {
      let _this = this
      this.fabricObj = new fabric.Canvas('canvas', {
        isDrawingMode: false,
        selectable: false,
        selection: false,
        devicePixelRatio: true, //Retina 高清屏 屏幕支持
      })
      this.fabricObj.freeDrawingBrush.color = '#E34F51'
      this.fabricObj.freeDrawingBrush.width = 2
      this.fabricObj.setWidth(this.canvasWidth)
      this.fabricObj.setHeight(500)
      this.fabricObj.add(
        new fabric.Rect({ top: 100, left: 100, width: 50, height: 50, fill: '#f55' }),
        new fabric.Circle({ top: 140, left: 230, radius: 75, fill: 'green' }),
        new fabric.Triangle({ top: 300, left: 210, width: 100, height: 100, fill: 'blue' }),
      );
      //绑定画板事件
      this.fabricObjAddEvent()
    },
    getX(o){
      return this.lastzoomPoint.x + (o.pointer.x - this.zoomPoint.x - this.relativeMouseX) / this.fabricObj.getZoom();
    },
    getY(o){
      return this.lastzoomPoint.y + (o.pointer.y - this.zoomPoint.y - this.relativeMouseY) / this.fabricObj.getZoom();
      
    },
    // 缩放后重置原点
    resetOriginAfterZoom(){
      this.lastzoomPoint.x = this.lastzoomPoint.x + (this.zoomPoint.x - this.lastmousePoint.x - this.relativeMouseX) / this.lastzoom
      this.lastzoomPoint.y = this.lastzoomPoint.y + (this.zoomPoint.y - this.lastmousePoint.y - this.relativeMouseY) / this.lastzoom

      this.lastmousePoint.x = this.zoomPoint.x
      this.lastmousePoint.y = this.zoomPoint.y
      this.lastzoom = this.zoom
    
      this.tempmouseX = this.relativeMouseX
      this.tempmouseY = this.relativeMouseY
      this.relativeMouseX = 0
      this.relativeMouseY = 0
    },
    //时间监听
    fabricObjAddEvent() {
      this.fabricObj.on({
        'mouse:down': (o) => {
          // this.mouseFrom.x = o.pointer.x;
          // this.mouseFrom.y = o.pointer.y;
          // 计算鼠标坐标 转换成相对画布坐标
          this.mouseFrom.x = this.getX(o)
          this.mouseFrom.y = this.getY(o)
          this.doDrawing = true;
          if (this.currentTool == 'text') {
            this.drawText()
          }
        },
        'mouse:up': (o) => {
          // 计算鼠标坐标 转换成相对画布坐标
          this.mouseTo.x = this.getX(o);
          this.mouseTo.y = this.getY(o);
          this.drawingObject = null;
          this.moveCount = 1;
          this.doDrawing = false;
          this.updateModifications(true);
        },
        'mouse:move': (o) => {
          if (this.moveCount % 2 && !this.doDrawing) {
            //减少绘制频率
            return;
          }
          // 画布平移 缩放后坐标全部都要转换  暂时没有实现转换方法 缩放后字体会变小 推荐新建画布
          // 画布平移功能 暂时关闭
          if(!o.target&&this.currentTool==='pick'){
            let delta = new fabric.Point(o.e.movementX, o.e.movementY);
            this.fabricObj.relativePan(delta);
            // 移动之后坐标处理
            this.relativeMouseX += o.e.movementX ;//累计每一次移动时候的偏差
            this.relativeMouseY += o.e.movementY ;
            // this.resetOrigin()
            return
          }
          this.moveCount++;
          // 计算鼠标坐标 转换成相对画布坐标
          this.mouseTo.x = this.getX(o);
          this.mouseTo.y = this.getY(o);
          this.drawing();
        },
        // 滚轮缩放功能 暂时关闭
        'mouse:wheel': (o) => {
          // let zoom = (o.e.deltaY > 0 ? -0.1 : 0.1) + this.fabricObj.getZoom();
          // zoom = Math.max(0.1, zoom); //最小为原来的1/10
          // zoom = Math.min(3, zoom); //最大是原来的3倍
          // let zoomPoint = new fabric.Point(o.e.pageX, o.e.pageY);
          // this.fabricObj.zoomToPoint(zoomPoint, zoom)
          this.zoom = (o.e.deltaY > 0 ? -0.1 : 0.1) + this.fabricObj.getZoom();
          this.zoom = Math.max(0.1, this.zoom); //最小为原来的1/10
          this.zoom = Math.min(3, this.zoom); //最大是原来的3倍
          this.zoomPoint = new fabric.Point(o.pointer.x,o.pointer.y);
          this.fabricObj.zoomToPoint(this.zoomPoint, this.zoom);
          this.resetOriginAfterZoom()
        },
        //对象移动时间
        'object:moving': (e) => {
          e.target.opacity = 0.5;
        },
        //增加对象
        'object:added': (e) => {
          // debugger
        },
        'object:modified': (e) => {
          e.target.opacity = 1;
          let object = e.target;
          this.updateModifications(true)
        },
        'selection:created': (e) => {
          if (this.currentTool !== 'remove') return
          if (e.target._objects) {
            //多选删除
            var etCount = e.target._objects.length;
            for (var etindex = 0; etindex < etCount; etindex++) {
              this.fabricObj.remove(e.target._objects[etindex]);
            }
          } else {
            //单选删除
            this.fabricObj.remove(e.target);
          }
          this.fabricObj.discardActiveObject(); //清楚选中框
          this.updateModifications(true)
        },
      });
    },
    //储存历史记录
    updateModifications(savehistory) {
      if (savehistory == true) {
        this.fabricHistoryJson.push(JSON.stringify(this.fabricObj))
      }
    },
    //canvas 历史后退
    undo() {
      let state = this.fabricHistoryJson
      if (this.mods < state.length) {
        this.fabricObj.clear().renderAll();
        this.fabricObj.loadFromJSON(state[state.length - 1 - this.mods - 1]);
        this.fabricObj.renderAll();
        this.mods += 1;
      }
    },
    //前进
    redo() {
      let state = this.fabricHistoryJson
      if (this.mods > 0) {
        this.fabricObj.clear().renderAll();
        this.fabricObj.loadFromJSON(state[state.length - 1 - this.mods + 1]);
        this.fabricObj.renderAll();
        this.mods -= 1;
      }
    },
    transformMouse(mouseX, mouseY) {
      return { x: mouseX / this.zoom, y: mouseY / this.zoom };
    },
    resetObj() {
      this.fabricObj.selectable = false
      this.fabricObj.selection = false
      this.fabricObj.skipTargetFind = true
      //清除文字对象
      if (this.textboxObj) {
        this.textboxObj.exitEditing();
        this.textboxObj = null;
      }
    },
    handleTools(tools, idx) {
      this.initIdx = idx;
      this.currentTool = tools.name;
      this.fabricObj.isDrawingMode = false;
      this.resetObj()
      switch (tools.name) {
        case 'pencil':
          this.fabricObj.isDrawingMode = true;
          break;
        case 'pick':
          // this.fabricObj.selection = true
          this.fabricObj.selectable = true
          this.fabricObj.skipTargetFind = false
          break
        case 'remove':
          this.fabricObj.selection = true
          this.fabricObj.selectable = true
          this.fabricObj.skipTargetFind = false
          break;
        case 'reset':
          this.fabricObj.clear();
          break;
        case 'redo':
          this.redo();
          break;
        case 'undo':
          this.undo();
          break;
        default:
          break;
      }
    },
    //绘制文字对象
    drawText() {
      this.textboxObj = new fabric.Textbox(" ", {
        left: this.mouseFrom.x,
        top: this.mouseFrom.y,
        width: 220,
        fontSize: 18,
        fill: this.drawColor,
        hasControls: true
      });
      this.fabricObj.add(this.textboxObj);
      this.textboxObj.enterEditing();
      this.textboxObj.hiddenTextarea.focus();
      this.updateModifications(true)
    },
    drawing() {
      let _this = this;
      if (this.drawingObject) {
        this.fabricObj.remove(this.drawingObject)
      }
      let fabricObject = null
      switch (this.currentTool) {
        case 'pencil':
          this.fabricObj.isDrawingMode = true
          break;
        case 'line':
          fabricObject = new fabric.Line([this.mouseFrom.x, this.mouseFrom.y, this.mouseTo.x, this.mouseTo.y], {
            stroke: this.drawColor,
            strokeWidth: this.drawWidth
          })
          break;
        case 'arrow':
          fabricObject = new fabric.Path(this.drawArrow(this.mouseFrom.x, this.mouseFrom.y, this.mouseTo.x, this.mouseTo.y, 17.5, 17.5), {
            stroke: this.drawColor,
            fill: "rgba(255,255,255,0)",
            strokeWidth: this.drawWidth
          });
          break;
        case 'xuxian': //doshed line
          fabricObject = new fabric.Line([this.mouseFrom.x, this.mouseFrom.y, this.mouseTo.x, this.mouseTo.y], {
            strokeDashArray: [10, 3],
            stroke: this.drawColor,
            strokeWidth: this.drawWidth
          })
          break;
        case 'juxing': //矩形
          let path = "M " +
            this.mouseFrom.x +
            " " +
            this.mouseFrom.y +
            " L " +
            this.mouseTo.x +
            " " +
            this.mouseFrom.y +
            " L " +
            this.mouseTo.x +
            " " +
            this.mouseTo.y +
            " L " +
            this.mouseFrom.x +
            " " +
            this.mouseTo.y +
            " L " +
            this.mouseFrom.x +
            " " +
            this.mouseFrom.y +
            " z";
          fabricObject = new fabric.Path(path, {
            left: Math.min(this.mouseFrom.x, this.mouseTo.x),
            top: Math.min(this.mouseFrom.y, this.mouseTo.y),
            stroke: this.drawColor,
            strokeWidth: this.drawWidth,
            fill: "rgba(255, 255, 255, 0)"
          });
          break;
        case "cricle": //正圆
          let radius = Math.sqrt((this.mouseTo.x - this.mouseFrom.x) * (this.mouseTo.x - this.mouseFrom.x) + (this.mouseTo.y - this.mouseFrom.y) * (this.mouseTo.y - this.mouseFrom.y)) / 2;
          fabricObject = new fabric.Circle({
            left: Math.min(this.mouseFrom.x, this.mouseTo.x),
            top: Math.min(this.mouseFrom.y, this.mouseTo.y),
            stroke: this.drawColor,
            fill: "rgba(255, 255, 255, 0)",
            radius: radius,
            strokeWidth: this.drawWidth
          });
          break;
        case "ellipse": //椭圆
          let left = this.mouseFrom.x;
          let top = this.mouseFrom.y;
          let ellipse = Math.sqrt((this.mouseTo.x - left) * (this.mouseTo.x - left) + (this.mouseTo.y - top) * (this.mouseTo.y - top)) / 2;
          fabricObject = new fabric.Ellipse({
            left: left,
            top: top,
            stroke: this.drawColor,
            fill: "rgba(255, 255, 255, 0)",
            originX: "center",
            originY: "center",
            rx: Math.abs(left - this.mouseTo.x),
            ry: Math.abs(top - this.mouseTo.y),
            strokeWidth: this.drawWidth
          });
          break;
        case "equilateral": //等边三角形
          let height = this.mouseTo.y - this.mouseFrom.y;
          fabricObject = new fabric.Triangle({
            left: Math.min(this.mouseFrom.x, this.mouseTo.x),
            top: Math.min(this.mouseFrom.y, this.mouseTo.y),
            width: Math.sqrt(Math.pow(height, 2) + Math.pow(height / 2.0, 2)),
            height: height,
            stroke: this.drawColor,
            strokeWidth: this.drawWidth,
            fill: "rgba(255,255,255,0)"
          });
          break;
        case "rightangle": //直角三角形
          let path2 = "M " + this.mouseFrom.x + " " + this.mouseFrom.y + " L " + this.mouseFrom.x + " " + this.mouseTo.y + " L " + this.mouseTo.x + " " + this.mouseTo.y + " z";
          fabricObject = new fabric.Path(path2, {
            left: Math.min(this.mouseFrom.x, this.mouseTo.x),
            top: Math.min(this.mouseFrom.y, this.mouseTo.y),
            stroke: this.drawColor,
            strokeWidth: this.drawWidth,
            fill: "rgba(255, 255, 255, 0)"
          });
          break;
        case 'remove':
          break;
        default:
          // statements_def'
          break;
      }
      if (fabricObject) {
        this.fabricObj.add(fabricObject)
        this.drawingObject = fabricObject;
      }
    },
    //书写箭头的方法
    drawArrow(fromX, fromY, toX, toY, theta, headlen) {
      theta = typeof theta != "undefined" ? theta : 30;
      headlen = typeof theta != "undefined" ? headlen : 10;
      // 计算各角度和对应的P2,P3坐标
      let angle = Math.atan2(fromY - toY, fromX - toX) * 180 / Math.PI,
        angle1 = (angle + theta) * Math.PI / 180,
        angle2 = (angle - theta) * Math.PI / 180,
        topX = headlen * Math.cos(angle1),
        topY = headlen * Math.sin(angle1),
        botX = headlen * Math.cos(angle2),
        botY = headlen * Math.sin(angle2);
      let arrowX = fromX - topX,
        arrowY = fromY - topY;
      let path = " M " + fromX + " " + fromY;
      path += " L " + toX + " " + toY;
      arrowX = toX + topX;
      arrowY = toY + topY;
      path += " M " + arrowX + " " + arrowY;
      path += " L " + toX + " " + toY;
      arrowX = toX + botX;
      arrowY = toY + botY;
      path += " L " + arrowX + " " + arrowY;
      return path;
    },
    downLoadImage() {
      this.done = true
      //生成双倍像素比的图片
      let base64URl = this.fabricObj.toDataURL({
        formart: 'png',
        multiplier: 2
      })
      this.imageBase64 = base64URl
      this.done = false
    },
  },
}

</script>
<style lang="scss" scoped>
.wraper {
  position: relative;
  width: 100%;
  height: 100%;

  .canvas-wraper {
    height: 50%;
    width: 100%;
    margin-bottom: 10px;
    overflow: hidden;
  }

  .controlPanel {
    width: 100%;
    height: 62px;
    background: #ddd;
    display: flex;
    justify-content: center;
    align-items: center;
    margin-bottom: 15px;

    .contro-item {
      flex-basis: 100px;
      border-right: 1px solid #dad7d9;
      text-align: center;
      cursor: pointer;
      background: #fefefe;

      i {
        font-size: 38px;
        line-height: 62px;
      }

      &.active {
        background: #e34f51;
        color: #fff;
        border-radius: 3px;

        i {
          font-size: 42px;
        }
      }
    }
  }

  .download {
    img {
      width: 100%;
    }
  }
}

</style>
