<template>
  <div class="item">
    <canvas id="c" width="1000" height="1000"> </canvas>
    <va-card>
      <va-card-block class="flex-nowrap">
        <va-card-title>防御塔信息</va-card-title>
        <va-card-content>
          等级：{{ targetTowerObj.level }} 攻击：{{
            Math.pow(2, targetTowerObj.level)
          }}
          范围：{{ targetTowerObj.level + 3 }}
        </va-card-content>
      </va-card-block>
      <va-card-block class="flex-nowrap">
        <va-card-title>蚂蚁信息</va-card-title>
        <va-card-content
          >体力：{{ targetAntObj.hp }}/{{ targetAntObj.maxHp }}
        </va-card-content>
      </va-card-block>
      <va-card-block class="flex-nowrap">
        <va-card-content>
          <h3 style="color: #000000">轮数:{{ round }}</h3>
          <h3 style="color: #900000">血量:{{ countR }}</h3>
          <h3 style="color: #900000">金币:{{ coinR }}</h3>
          <h3 style="color: #009000">血量:{{ countG }}</h3>
          <h3 style="color: #009000">金币:{{ coinG }}</h3>
          <h3 style="color: #000090">{{ endMsg }}</h3>
          <va-file-upload
            @file-added="fileAdded"
            v-model="recordFile"
            upload-button-text="上传回放"
          />
          <va-button @click="autoRound"> 自动播放 </va-button>
          <va-button @click="oneRound" :disabled="cooldown">
            进行一轮
          </va-button>
        </va-card-content>
      </va-card-block>
    </va-card>
    <div class="details" margin-left="800" margin-top="200">
      <img id="redTowerImg" src="/tower1.png" />
      <img id="greenTowerImg" src="/tower3.png" />
      <img id="redImg" src="/targetAnt.png" />
      <img id="greenImg" src="/greenAnt.png" />
      <img id="castleImg" src="/castle.png" />
      <img id="holeImg" src="/hole.png" />
      <img id="itemImg" src="/lemon.png" />
    </div>
  </div>
</template>

<script lang="ts">
import { fabric } from "fabric";
var canvas;
var nullAnt = { hp: NaN, maxHp: NaN };
var nullTower = { level: NaN };
var interval;

export default {
  data() {
    return {
      countR: 10,
      coinR: 10,
      countG: 10,
      coinG: 10,
      round: 0,
      recordFile: [],
      fileContent: [],
      tmpRecordLine: 0,
      towers: [],
      ants: [],
      pheromone: [],
      numAnt: 0,
      numTower: 0,
      targetX: -1,
      targetY: -1,
      watchingAntId: -1,
      watchingTowerId: -1,
      cantBuild: true,
      targetAnt: -1,
      targetTower: -1,
      cooldown: false,
      targetAntObj: nullAnt,
      targetTowerObj: nullTower,
      autoPlay: false,
      antSet: new Set(),
      endMsg: "",
      showModal: false,
      beginHP: 10,
      rateHP: 1.005,
      minPhe: 0.001,
      basePhe: 0.5,
      decayRate: 0.7,
      pathAdd: 0,
      goalAdd: 100,
      deathAdd: -50,
      oldAdd: -100,
    };
  },

  watch: {
    autoPlay(newone, oldone) {
      if (newone) {
        this.cooldown = true;
        var TMPthis = this;
        interval = setInterval(function () {
          if (TMPthis.tmpRecordLine >= TMPthis.fileContent.length) {
            clearInterval(interval);
            this.endMsg = "游戏结束";
          } else TMPthis.oneRound();
        }, 500);
      } else {
        this.cooldown = false;
        clearInterval(interval);
      }
    },
  },

  mounted() {
    canvas = new fabric.Canvas("c");
    canvas.set("centeredRotation", true);

    var X: number;
    var Y: number;
    // init canvas
    for (X = 0; X <= 22; X++) {
      this.pheromone.push([]);
      for (Y = 0; Y <= 22; Y++) {
        this.pheromone[X].push([]);
        if (
          Y < 11 - (2 * X + 1) ||
          Y > 11 + (2 * X + 1) ||
          Y < 2 * X - 33 ||
          Y > 55 - 2 * X
        ) {
          continue;
        }
        var p: number;
        var dir: number[][];
        if (!(Y % 2))
          dir = [
            [-1, 0],
            [0, 1],
            [1, 1],
            [1, 0],
            [1, -1],
            [0, -1],
          ];
        else
          dir = [
            [-1, 0],
            [-1, 1],
            [0, 1],
            [1, 0],
            [0, -1],
            [-1, -1],
          ];
        for (p = 0; p < 6; p++) {
          dir[p][0] += X;
          dir[p][1] += Y;
          if (
            dir[p][1] < 11 - (2 * dir[p][0] + 1) ||
            dir[p][1] > 11 + (2 * dir[p][0] + 1) ||
            dir[p][1] < 2 * dir[p][0] - 33 ||
            dir[p][1] > 55 - 2 * dir[p][0] ||
            dir[p][1] < 0 ||
            dir[p][1] > 22
          )
            this.pheromone[X][Y].push(0);
          else this.pheromone[X][Y].push(1);
        }

        var Location: number[];
        if (!(Y % 2))
          Location = [30 * Y, 20 * Math.sqrt(3) * X + 10 * Math.sqrt(3)];
        else Location = [30 * Y, 20 * Math.sqrt(3) * X];
        var points: number[] = [
          {
            x: 10,
            y: 0,
          },
          {
            x: 30,
            y: 0,
          },
          {
            x: 40,
            y: 10 * Math.sqrt(3),
          },
          {
            x: 30,
            y: 20 * Math.sqrt(3),
          },
          {
            x: 10,
            y: 20 * Math.sqrt(3),
          },
          {
            x: 0,
            y: 10 * Math.sqrt(3),
          },
        ];
        var polygon = new fabric.Polygon(points, {
          left: 100 + Location[0],
          top: 30 + Location[1],
          fill: "yellow",
          strokeWidth: 2,
          stroke: "cyan",
          scaleX: 1,
          scaleY: 1,
          objectCaching: false,
          hoverCursor: "pointer",
        });
        polygon.set("selectable", false);
        polygon.name = ("P_" + X + "_" + Y) as string;
        if (X < 6 || (X == 16 && Y % 2 == 0) || X > 16)
          polygon.set("fill", "#EEEEEE");
        canvas.add(polygon);
      }
    }

    var Location: number[];
    Location = [30 * 11, 20 * Math.sqrt(3) * 20];

    var imgElement = document.getElementById("castleImg");
    var imgInstance = new fabric.Image(imgElement, {
      left: 120 + Location[0],
      top: 30 + Location[1] + 10 * Math.sqrt(3),
      originX: "center",
      originY: "center",
      scaleX: 0.5,
      scaleY: 0.5,
      selectable: false,
      objectCaching: false,
    });
    imgInstance.name = "H";
    canvas.add(imgInstance);
    canvas.bringForward(imgInstance);
    setTimeout(
      function (a) {
        a.animate("opacity", 1, {
          duration: 350,
          onChange: canvas.renderAll.bind(canvas),
          easing: fabric.util.ease.easeInCubic,
        });
      },
      10,
      imgInstance
    );

    Location = [30 * 11, 20 * Math.sqrt(3) * 2];

    var imgElement = document.getElementById("castleImg");
    var imgInstance = new fabric.Image(imgElement, {
      left: 120 + Location[0],
      top: 30 + Location[1] + 10 * Math.sqrt(3),
      originX: "center",
      originY: "center",
      scaleX: 0.5,
      scaleY: 0.5,
      selectable: false,
      objectCaching: false,
    });
    imgInstance.name = "C";
    canvas.add(imgInstance);

    canvas.bringForward(imgInstance);
    setTimeout(
      function (a) {
        a.animate("opacity", 1, {
          duration: 350,
          onChange: canvas.renderAll.bind(canvas),
          easing: fabric.util.ease.easeInCubic,
        });
      },
      10,
      imgInstance
    );

    canvas.renderAll();

    var TMPthis = this;

    canvas.on("mouse:down", function (options) {
      if (options.target) {
        console.log("一个格子被点击了!", options.target.name);

        if (options.target.name[0] == "P") {
          var posLst = options.target.name.split("_");
          console.log(
            "信息素",
            TMPthis.pheromone[posLst[1]][posLst[2]].map((x) => {
              return x + (TMPthis.basePhe as number);
            })
          );
          if (posLst[1] == TMPthis.targetX && posLst[2] == TMPthis.targetY) {
            options.target.set("fill", "yellow");
            TMPthis.targetX = TMPthis.targetY = -1;
          } else if (options.target.fill == "yellow") {
            canvas
              .getObjects()
              .filter((element) => {
                return (
                  element.name == "P_" + TMPthis.targetX + "_" + TMPthis.targetY
                );
              })
              .forEach((element) => {
                element.set("fill", "yellow");
              });
            options.target.set("fill", "orange");
            TMPthis.targetX = posLst[1] as number;
            TMPthis.targetY = posLst[2] as number;
          }
        }

        if (options.target.name[0] == "A") {
          if (TMPthis.targetAnt != options.target.name.slice(2)) {
            var imgElement = document.getElementById("antImg");
            if (TMPthis.targetAntObj.hp != NaN) {
              var obj = canvas.getObjects().filter((element) => {
                return element.name == "A_" + TMPthis.targetAntObj.id;
              })[0];
              if (obj != undefined) {
                var imgInstance = new fabric.Image(imgElement, {
                  left: obj.left,
                  top: obj.top,
                  angle: obj.angle,
                  originX: "center",
                  originY: "center",
                  scaleX: 0.7,
                  scaleY: 0.7,
                  selectable: false,
                });
                imgInstance.name = obj.name;
                canvas.remove(obj);
                canvas.add(imgInstance);
              }
            }

            var imgElement = document.getElementById("targetImg");
            var imgInstance = new fabric.Image(imgElement, {
              left: options.target.left,
              top: options.target.top,
              angle: options.target.angle,
              originX: "center",
              originY: "center",
              scaleX: 0.7,
              scaleY: 0.7,
              selectable: false,
            });
            imgInstance.name = options.target.name;
            canvas.remove(options.target);
            canvas.add(imgInstance);
            TMPthis.targetAnt = imgInstance.name.slice(2) as number;
            TMPthis.targetAntObj = TMPthis.ants.filter((element) => {
              return element.id == TMPthis.targetAnt;
            })[0];
            if (!TMPthis.targetAntObj) {
              TMPthis.targetAntObj = nullAnt;
              canvas.remove(imgInstance);
            }
          } else {
            var imgElement = document.getElementById("antImg");
            var imgInstance = new fabric.Image(imgElement, {
              left: options.target.left,
              top: options.target.top,
              angle: options.target.angle,
              originX: "center",
              originY: "center",
              scaleX: 0.7,
              scaleY: 0.7,
              selectable: false,
            });
            imgInstance.name = options.target.name;
            canvas.remove(options.target);
            canvas.add(imgInstance);
            TMPthis.targetAnt = -1;
            TMPthis.targetAntObj = nullAnt;
          }
        }

        if (options.target.name[0] == "T") {
          TMPthis.targetTower = options.target.name.slice(2) as number;
          TMPthis.targetTowerObj = TMPthis.towers.filter((element) => {
            return element.id == TMPthis.targetTower;
          })[0];
        }
      }
    });
    canvas.renderAll();
  },
  methods: {
    fileAdded() {
      let file = this.recordFile[0];
      // 检测浏览器对FileReader的支持
      if (window.FileReader) {
        const reader = new FileReader();
        var TMPthis = this;
        reader.onload = function (e) {
          console.log("读取成功", e);
          TMPthis.fileContent = e.target.result.split("\n");
          TMPthis.tmpRecordLine = 0;
          console.log(TMPthis.fileContent);
        };
        reader.readAsText(file, "utf-8");
      } else {
        alert("Not supported by your browser!");
      }
    },

    autoRound() {
      this.autoPlay = !this.autoPlay;
    },

    oneRound() {
      if (this.autoPlay == false) {
        this.cooldown = true;
        setTimeout(() => {
          this.cooldown = false;
        }, 1000);
      }
      this.round = this.fileContent[this.tmpRecordLine] as number;
      this.tmpRecordLine++;
      var coins = this.fileContent[this.tmpRecordLine].split(" ");
      this.coinR = coins[0] as number;
      this.coinG = coins[1] as number;
      this.tmpRecordLine++;
      var HPs = this.fileContent[this.tmpRecordLine].split(" ");
      console.log(this.tmpRecordLine);
      this.countR = HPs[0] as number;
      this.countG = HPs[1] as number;
      this.tmpRecordLine++;
      var Barracks = this.fileContent[this.tmpRecordLine] as number;
      this.renderBarrack(++this.tmpRecordLine, Barracks);
      console.log(this.tmpRecordLine);
      this.tmpRecordLine += (Barracks as number) * 1;
      var Ants = this.fileContent[this.tmpRecordLine] as number;
      this.renderAnt(++this.tmpRecordLine, Ants);
      console.log(this.tmpRecordLine);
      this.tmpRecordLine += (Ants as number) * 1;
      var Towers = this.fileContent[this.tmpRecordLine] as number;
      this.renderTower(++this.tmpRecordLine, Towers);
      console.log(this.tmpRecordLine);
      this.tmpRecordLine += (Towers as number) * 1;
      var Items = this.fileContent[this.tmpRecordLine] as number;
      this.renderItem(++this.tmpRecordLine, Items);
      console.log(this.tmpRecordLine);
      this.tmpRecordLine += (Items as number) * 1;

      var AppliedItems = this.fileContent[this.tmpRecordLine] as number;
      console.log(
        this.fileContent.slice(
          this.tmpRecordLine + 1,
          this.tmpRecordLine + AppliedItems
        )
      );
      this.tmpRecordLine += (AppliedItems as number) * 1;
      //this.decayPheromon();
      canvas.renderAll();
      this.tmpRecordLine++;
    },

    renderBarrack(lineIdx, num) {
      canvas
        .getObjects()
        .filter((element) => {
          return element.name[0] === "B";
        })
        .forEach((element) => {
          canvas.remove(element);
        });
      while (num--) {
        var newline = this.fileContent[lineIdx].split(" ");
        var newBarrack: Barrack = {
          id: newline[0] as number,
          X: newline[1] as number,
          Y: newline[2] as number,
        };
        if (!(newBarrack.Y % 2))
          Location = [
            30 * newBarrack.Y,
            20 * Math.sqrt(3) * newBarrack.X + 10 * Math.sqrt(3),
          ];
        else Location = [30 * newBarrack.Y, 20 * Math.sqrt(3) * newBarrack.X];
        var imgElement = document.getElementById("holeImg");
        var imgInstance = new fabric.Image(imgElement, {
          left: 120 + Location[0],
          top: 30 + Location[1] + 10 * Math.sqrt(3),
          originX: "center",
          originY: "center",
          scaleX: 0.5,
          scaleY: 0.5,
          selectable: false,
        });
        imgInstance.name = "B_" + (newBarrack.id as string);

        canvas.add(imgInstance);
        canvas.renderAll();
        lineIdx++;
      }
    },

    renderAnt(lineIdx, num) {
      canvas
        .getObjects()
        .filter((element) => {
          return element.name[0] === "A";
        })
        .forEach((element) => {
          canvas.remove(element);
        });
      while (num--) {
        var newline = this.fileContent[lineIdx].split(" ");
        var newAnt: Ant = {
          id: newline[0] as number,
          X: newline[1] as number,
          Y: newline[2] as number,
          player: newline[3] as number,
          hp: newline[4] as number,
          state: newline[5] as number,
        };
        if (!(newAnt.Y % 2))
          Location = [
            30 * newAnt.Y,
            20 * Math.sqrt(3) * newAnt.X + 10 * Math.sqrt(3),
          ];
        else Location = [30 * newAnt.Y, 20 * Math.sqrt(3) * newAnt.X];
        if (newAnt.player == 1)
          var imgElement = document.getElementById("redImg");
        if (newAnt.player == 0)
          var imgElement = document.getElementById("greenImg");
        var imgInstance = new fabric.Image(imgElement, {
          left: 120 + Location[0],
          top: 30 + Location[1] + 10 * Math.sqrt(3),
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = "A_" + (newAnt.id as string);

        canvas.add(imgInstance);
        if (newAnt.player == 0) imgInstance.rotate(180);
        canvas.renderAll();
        lineIdx++;
      }
    },

    renderTower(lineIdx, num) {
      canvas
        .getObjects()
        .filter((element) => {
          return element.name[0] === "T";
        })
        .forEach((element) => {
          canvas.remove(element);
        });
      while (num--) {
        var newline = this.fileContent[lineIdx].split(" ");
        var newTower: Tower = {
          id: newline[0] as number,
          X: newline[1] as number,
          Y: newline[2] as number,
          player: newline[3] as number,
          type: newline[4] as number,
        };
        if (!(newTower.Y % 2))
          Location = [
            30 * newTower.Y,
            20 * Math.sqrt(3) * newTower.X + 10 * Math.sqrt(3),
          ];
        else Location = [30 * newTower.Y, 20 * Math.sqrt(3) * newTower.X];
        if (newTower.player == 1)
          var imgElement = document.getElementById("redTowerImg");
        if (newTower.player == 0)
          var imgElement = document.getElementById("greenTowerImg");
        var imgInstance = new fabric.Image(imgElement, {
          left: 120 + Location[0],
          top: 30 + Location[1] + 10 * Math.sqrt(3),
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = "T_" + (newTower.id as string);

        canvas.add(imgInstance);
        if (newTower.player == 0) imgInstance.rotate(180);
        canvas.renderAll();
        lineIdx++;
      }
    },

    renderItem(lineIdx, num) {
      canvas
        .getObjects()
        .filter((element) => {
          return element.name[0] === "I";
        })
        .forEach((element) => {
          canvas.remove(element);
        });
      while (num--) {
        var newline = this.fileContent[lineIdx].split(" ");
        var newItem: Item = {
          id: newline[0] as number,
          X: newline[1] as number,
          Y: newline[2] as number,
          type: newline[3] as number,
        };
        if (!(newItem.Y % 2))
          Location = [
            30 * newItem.Y,
            20 * Math.sqrt(3) * newItem.X + 10 * Math.sqrt(3),
          ];
        else Location = [30 * newItem.Y, 20 * Math.sqrt(3) * newItem.X];
        var imgElement = document.getElementById("itemImg");
        var imgInstance = new fabric.Image(imgElement, {
          left: 120 + Location[0],
          top: 30 + Location[1] + 10 * Math.sqrt(3),
          originX: "center",
          originY: "center",
          scaleX: 0.5,
          scaleY: 0.5,
          selectable: false,
        });
        imgInstance.name = "I_" + (newItem.id as string);

        canvas.add(imgInstance);
        canvas.renderAll();
        lineIdx++;
      }
    },
  },
};
</script>

<style scoped>
.item {
  margin-top: 2rem;
  display: flex;
}

.details {
  flex: 1;
  margin-left: 1rem;
}

i {
  display: flex;
  place-items: center;
  place-content: center;
  width: 32px;
  height: 32px;

  color: var(--color-text);
}

h3 {
  font-size: 1.2rem;
  font-weight: 500;
  margin-bottom: 0.4rem;
  color: var(--color-heading);
}

@media (min-width: 1024px) {
  .item {
    margin-top: 0;
    padding: 0.4rem 0 1rem calc(var(--section-gap) / 2);
  }

  i {
    top: calc(50% - 25px);
    left: -26px;
    position: absolute;
    border: 1px solid var(--color-border);
    background: var(--color-background);
    border-radius: 8px;
    width: 50px;
    height: 50px;
  }

  .item:before {
    content: " ";
    border-left: 1px solid var(--color-border);
    position: absolute;
    left: 0;
    bottom: calc(50% + 25px);
    height: calc(50% - 25px);
  }

  .item:after {
    content: " ";
    border-left: 1px solid var(--color-border);
    position: absolute;
    left: 0;
    top: calc(50% + 25px);
    height: calc(50% - 25px);
  }

  .item:first-of-type:before {
    display: none;
  }

  .item:last-of-type:after {
    display: none;
  }
}
</style>
