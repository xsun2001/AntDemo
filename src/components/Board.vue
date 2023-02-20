<template>
  <div class="item">
    <canvas id="c" width="1000" height="1000"> </canvas>
    <va-card>
      <va-card-block class="flex-nowrap">
        <va-card-title>防御塔信息</va-card-title>
        <va-card-content>
          等级：{{ targetTowerObj.level }} 攻击：{{
            targetTowerObj.level * 2
          }}
          范围：{{ targetTowerObj.level + 3 }}
        </va-card-content>
        <va-card-actions align="between">
          <va-button
            @click="upgrade"
            :disabled="
              (targetTowerObj.level == 1 && coin < 40) ||
              (targetTowerObj.level == 2 && coin < 150) ||
              targetTowerObj.level == 3 ||
              isNaN(targetTowerObj.level)
            "
            >升级</va-button
          >
          <va-button @click="downgrade" :disabled="isNaN(targetTowerObj.level)"
            >降级</va-button
          >
        </va-card-actions>
      </va-card-block>
      <va-card-block class="flex-nowrap">
        <va-card-title>蚂蚁信息</va-card-title>
        <va-card-content
          >体力：{{ targetAntObj.hp }}/{{ targetAntObj.maxHp }}
        </va-card-content>
      </va-card-block>
      <va-card-block class="flex-nowrap">
        <va-card-content>
          <h3 style="color: #003c76">血量:{{ count }}</h3>
          <h3 style="color: #003c76">金币:{{ coin }}</h3>
          <va-button @click="oneRound" :disabled="cooldown">
            进行一轮
          </va-button>
          <va-button @click="buildTower" :disabled="cantBuildTower">
            建立防御塔
          </va-button>
          <va-button @click="autoRound"> 快进 </va-button>
          <h3 style="color: #003c76">需要:{{ buildCost }}金币</h3>
          <h3 style="color: #d30000">{{ endMsg }}</h3>
        </va-card-content>
      </va-card-block>
    </va-card>
    <div class="details" margin-left="800" margin-top="200">
      <img id="antImg" src="src/assets/ant.png" />
      <img id="towerImg1" src="src/assets/tower1.png" />
      <img id="towerImg2" src="src/assets/tower2.png" />
      <img id="towerImg3" src="src/assets/tower3.png" />
      <img id="targetImg" src="src/assets/targetAnt.png" />
      <img id="holeImg" src="src/assets/hole.png" />
      <img id="castleImg" src="src/assets/castle.png" />
      <va-button @click="showModal = !showModal"> 参数设置 </va-button>
      <va-modal v-model="showModal" title="参数设置">
        <va-input class="mb-6" v-model="beginHP" label="初始蚂蚁血量" />
        <va-input class="mb-6" v-model="rateHP" label="血量增加率" />
        <va-input class="mb-6" v-model="pathAdd" label="每步信息素" />
        <va-input class="mb-6" v-model="goalAdd" label="到达信息素" />
        <va-input class="mb-6" v-model="deathAdd" label="死亡信息素" />
        <va-input class="mb-6" v-model="oldAdd" label="老死信息素" />
        <va-input class="mb-6" v-model="decayRate" label="衰减率" />
        <va-input class="mb-6" v-model="minPhe" label="最小信息素" />
      </va-modal>
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
      count: 10,
      coin: 10,
      round: 0,
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
      rateHP: 1.01,
      minPhe: 0.001,
      decayRate: 0.9,
      pathAdd: 1,
      goalAdd: 100,
      deathAdd: -50,
      oldAdd: -100,
    };
  },

  computed: {
    buildCost(): number {
      return Math.floor(10 * Math.pow(1.5, this.towers.length));
    },
    newAntHP(): number {
      return Math.floor(
        (this.beginHP as number) * Math.pow(this.rateHP as number, this.round)
      );
    },
    cantBuildTower(): boolean {
      return this.coin < this.buildCost || this.targetX === -1;
    },
  },

  watch: {
    autoPlay(newone, oldone) {
      if (newone) {
        this.cooldown = true;
        var TMPthis = this;
        interval = setInterval(function () {
          if (this.count <= 0) {
            clearInterval(interval);
            this.endMsg = "你失败了！\n坚持了" + this.round + "轮";
          }
          TMPthis.oneRound();
        }, 1000);
      } else {
        this.cooldown = false;
        clearInterval(interval);
      }
    },
    count(newone, oldone) {
      if (newone <= 0) {
        clearInterval(interval);
        this.cooldown = true;
        this.endMsg = "你失败了！\n坚持了" + this.round + "轮";
      }
    },
  },

  mounted() {
    canvas = new fabric.Canvas("c");
    canvas.set("centeredRotation", true);

    var X: number;
    var Y: number;
    // init canvas
    for (X = 0; X <= 20; X++) {
      this.pheromone.push([]);
      for (Y = 0; Y <= 20; Y++) {
        this.pheromone[X].push([]);
        if (
          Y < 10 - (2 * X + 1) ||
          Y > 10 + (2 * X + 1) ||
          Y < 2 * X - 30 ||
          Y > 50 - 2 * X
        ) {
          continue;
        }
        var p: number;
        var dir: number[][];
        if (Y % 2)
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
            dir[p][1] < 10 - (2 * dir[p][0] + 1) ||
            dir[p][1] > 10 + (2 * dir[p][0] + 1) ||
            dir[p][1] < 2 * dir[p][0] - 30 ||
            dir[p][1] > 50 - 2 * dir[p][0] ||
            dir[p][1] < 0 ||
            dir[p][1] > 20
          )
            this.pheromone[X][Y].push(0);
          else this.pheromone[X][Y].push(1);
        }

        var Location: number[];
        if (Y % 2)
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
        if (X >= 15) polygon.set("fill", "#EEEEEE");
        canvas.add(polygon);
      }
    }

    var Location: number[];
    Location = [30 * 10, 20 * Math.sqrt(3) * 18];

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
    imgInstance.name = "H";
    canvas.add(imgInstance);

    Location = [30 * 10, 20 * Math.sqrt(3) * 2];

    var imgElement = document.getElementById("castleImg");
    var imgInstance = new fabric.Image(imgElement, {
      left: 120 + Location[0],
      top: 30 + Location[1] + 10 * Math.sqrt(3),
      originX: "center",
      originY: "center",
      scaleX: 0.5,
      scaleY: 0.5,
      selectable: false,
    });
    imgInstance.name = "C";
    canvas.add(imgInstance);

    var TMPthis = this;

    canvas.on("mouse:down", function (options) {
      if (options.target) {
        console.log("一个格子被点击了!", options.target.name);

        if (options.target.name[0] == "P") {
          var posLst = options.target.name.split("_");
          console.log("信息素", TMPthis.pheromone[posLst[1]][posLst[2]]);
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
      this.coin++;
      if (this.round % 4 == 0) this.generateAnt();
      var x: number;
      if (this.round % 2 == 0)
        for (x = 0; x < this.towers.length; x++)
          this.towerAttack(this.towers[x]);
      for (x = 0; x < this.ants.length; x++) {
        if (this.ants[x].hp <= 0) {
          this.coin += Math.floor(Math.sqrt(this.ants[x].maxHp - 1)) + 1;
          this.routePheromone(this.ants[x], this.deathAdd as number);
        } else if (this.ants[x].path.length >= 40) {
          this.ants[x].hp = 0;
          this.routePheromone(this.ants[x], this.oldAdd as number);
        } else {
          this.antMove(this.ants[x]);
          if (this.ants[x].X == 2 && this.ants[x].Y == 10) {
            this.count--;
            this.ants[x].hp = 0;
            this.routePheromone(this.ants[x], this.goalAdd as number);
          }
        }
      }
      if (this.targetAntObj && this.targetAntObj.hp <= 0)
        this.targetAntObj = nullAnt;
      var TMPthis = this;
      for (x = 0; x < TMPthis.ants.length; x++) {
        var tmpAnt = TMPthis.ants[x];
        if (tmpAnt.hp <= 0 || isNaN(tmpAnt.hp)) {
          TMPthis.ants = [
            ...TMPthis.ants.slice(0, x),
            ...TMPthis.ants.slice(x + 1),
          ];
          x = 0;
          var ID = tmpAnt.id;
          if (this.antSet.has(ID)) {
            this.antSet.delete(ID);
            setTimeout(
              (a) => {
                canvas.remove(
                  canvas.getObjects().filter((element) => {
                    return element.name === "A_" + a;
                  })[0]
                );
              },
              550,
              ID
            );
          }
        }
      }

      this.decayPheromon();
      canvas.renderAll();
      this.round++;
    },

    generateAnt() {
      var newAnt: Ant = {
        id: this.numAnt,
        X: 18,
        Y: 10,
        lastStep: -1,
        maxHp: this.newAntHP,
        hp: this.newAntHP,
        path: [],
      };
      this.ants.push(newAnt);
      this.antSet.add(this.numAnt);
      Location = [30 * 10, 20 * Math.sqrt(3) * 18];

      var imgElement = document.getElementById("antImg");
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
      canvas.renderAll();

      this.numAnt++;
    },

    antMove(theAnt: Ant) {
      var dir: number[][];
      var prob: number[] = [];
      var totProb: number = 0;
      if (theAnt.Y % 2)
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
      var x: number;
      for (x = 0; x < 6; x++) {
        dir[x][0] += theAnt.X;
        dir[x][1] += theAnt.Y;
        if (dir[x][0] == 2 && dir[x][1] == 10) {
          prob = [0, 0, 0, 0, 0, 0];
          prob[x] = 1;
          totProb = 1;
          break;
        } else if (
          dir[x][1] < 10 - (2 * dir[x][0] + 1) ||
          dir[x][1] > 10 + (2 * dir[x][0] + 1) ||
          dir[x][1] < 2 * dir[x][0] - 30 ||
          dir[x][1] > 50 - 2 * dir[x][0] ||
          dir[x][1] < 0 ||
          dir[x][1] > 20 ||
          x - theAnt.lastStep == 3 ||
          x - theAnt.lastStep == -3
        )
          prob.push(0);
        else {
          var tmp = this.pheromone[theAnt.X][theAnt.Y][x];
          if (x == 0) tmp *= 3;
          if (x == 1 || x == 5) tmp *= 2;
          prob.push(tmp);
          totProb += tmp;
        }
      }
      var ans = Math.random() * totProb;
      for (x = 0; x < 6; x++) {
        ans -= prob[x];
        if (ans < 0) {
          theAnt.lastStep = x;
          theAnt.path.push([theAnt.X, theAnt.Y, x]);
          this.pheromone[theAnt.X][theAnt.Y][x] += (this.pathAdd as number) * 1;

          theAnt.X = dir[x][0];
          theAnt.Y = dir[x][1];

          if (theAnt.Y % 2)
            Location = [
              30 * theAnt.Y,
              20 * Math.sqrt(3) * theAnt.X + 10 * Math.sqrt(3),
            ];
          else Location = [30 * theAnt.Y, 20 * Math.sqrt(3) * theAnt.X];
          var A = Location[0];
          var B = Location[1];

          canvas
            .getObjects()
            .filter((element) => {
              return element.name == "A_" + theAnt.id;
            })
            .forEach((element) => {
              // 旋转动画
              setTimeout(
                function (a, b) {
                  a.animate("left", b + 120, {
                    duration: 350,
                    onChange: canvas.renderAll.bind(canvas),
                    easing: fabric.util.ease.easeInCubic,
                  });
                },
                600,
                element,
                A
              );
              setTimeout(
                function (a, b) {
                  a.animate("top", 30 + b + 10 * Math.sqrt(3), {
                    duration: 350,
                    onChange: canvas.renderAll.bind(canvas),
                    easing: fabric.util.ease.easeInCubic,
                  });
                },
                600,
                element,
                B
              );
              element.rotate(60 * x);
            });

          canvas.renderAll();
          break;
        }
      }
    },

    buildTower() {
      this.coin -= this.buildCost;
      var newTower: Tower = {
        id: this.numTower,
        X: this.targetX,
        Y: this.targetY,
        level: 1,
      };
      this.towers.push(newTower);
      if (this.targetY % 2)
        Location = [
          30 * this.targetY,
          20 * Math.sqrt(3) * this.targetX + 10 * Math.sqrt(3),
        ];
      else Location = [30 * this.targetY, 20 * Math.sqrt(3) * this.targetX];

      var imgElement = document.getElementById("towerImg1");
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
      var TMPthis = this;
      canvas
        .getObjects()
        .filter((element) => {
          return element.name == "P_" + TMPthis.targetX + "_" + TMPthis.targetY;
        })
        .forEach((element) => {
          element.set("fill", "yellow");
        });
      canvas.renderAll();

      this.targetX = this.targetY = -1;
      this.numTower++;
    },

    upgrade() {
      if (this.targetTowerObj.level == 1) {
        this.coin -= 40;
        var imgElement = document.getElementById("towerImg2");

        var oldElement = canvas.getObjects().filter((element) => {
          return element.name == "T_" + this.targetTower;
        })[0];

        var imgInstance = new fabric.Image(imgElement, {
          left: oldElement.left,
          top: oldElement.top,
          angle: oldElement.angle,
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = oldElement.name;
        canvas.remove(oldElement);
        canvas.add(imgInstance);
        this.targetTowerObj.level = 2;
      } else if (this.targetTowerObj.level == 2) {
        this.coin -= 150;
        var imgElement = document.getElementById("towerImg3");

        var oldElement = canvas.getObjects().filter((element) => {
          return element.name == "T_" + this.targetTower;
        })[0];

        var imgInstance = new fabric.Image(imgElement, {
          left: oldElement.left,
          top: oldElement.top,
          angle: oldElement.angle,
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = oldElement.name;
        canvas.remove(oldElement);
        canvas.add(imgInstance);
        this.targetTowerObj.level = 3;
      }
    },

    downgrade() {
      if (this.targetTowerObj.level == 3) {
        this.coin += 75;
        var imgElement = document.getElementById("towerImg2");

        var oldElement = canvas.getObjects().filter((element) => {
          return element.name == "T_" + this.targetTower;
        })[0];

        var imgInstance = new fabric.Image(imgElement, {
          left: oldElement.left,
          top: oldElement.top,
          angle: oldElement.angle,
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = oldElement.name;
        canvas.remove(oldElement);
        canvas.add(imgInstance);
        this.targetTowerObj.level = 2;
      } else if (this.targetTowerObj.level == 2) {
        this.coin += 20;
        var imgElement = document.getElementById("towerImg1");

        var oldElement = canvas.getObjects().filter((element) => {
          return element.name == "T_" + this.targetTower;
        })[0];

        var imgInstance = new fabric.Image(imgElement, {
          left: oldElement.left,
          top: oldElement.top,
          angle: oldElement.angle,
          originX: "center",
          originY: "center",
          scaleX: 0.7,
          scaleY: 0.7,
          selectable: false,
        });
        imgInstance.name = oldElement.name;
        canvas.remove(oldElement);
        canvas.add(imgInstance);
        this.targetTowerObj.level = 1;
      } else if (this.targetTowerObj.level == 1) {
        this.towers = this.towers.filter((element) => {
          return element.id != this.targetTower;
        });

        this.coin += Math.floor(this.buildCost * 0.5);

        var oldElement = canvas.getObjects().filter((element) => {
          return element.name === "T_" + this.targetTower;
        })[0];

        canvas.remove(oldElement);

        this.targetTowerObj = nullTower;
        this.targetTower = -1;
      }
    },

    towerAttack(theTower: Tower) {
      var damage = theTower.level * 2;
      var range = 3 + theTower.level;
      var vis: number[][] = [];
      var X: number;
      var Y: number;
      for (X = 0; X <= 20; X++) {
        vis.push([]);
        for (Y = 0; Y <= 20; Y++) {
          if (X == theTower.X && Y == theTower.Y) vis[X].push(0);
          else vis[X].push(-1);
        }
      }
      var r: number;
      for (r = 0; r < range; r++) {
        for (X = 0; X <= 20; X++)
          for (Y = 0; Y <= 20; Y++) {
            if (
              Y < 10 - (2 * X + 1) ||
              Y > 10 + (2 * X + 1) ||
              Y < 2 * X - 30 ||
              Y > 50 - 2 * X ||
              vis[X][Y] != r
            ) {
              continue;
            }
            var dir: number[][];
            if (Y % 2)
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
            var p: number;
            for (p = 0; p < 6; p++) {
              if (
                Y + dir[p][1] < 10 - (2 * (X + dir[p][0]) + 1) ||
                Y + dir[p][1] > 10 + (2 * (X + dir[p][0]) + 1) ||
                Y + dir[p][1] < 2 * (X + dir[p][0]) - 30 ||
                Y + dir[p][1] > 50 - 2 * (X + dir[p][0])
              )
                continue;
              if (vis[X + dir[p][0]][Y + dir[p][1]] == -1)
                vis[X + dir[p][0]][Y + dir[p][1]] = r + 1;
            }
          }
      }
      var goal: Ant = nullAnt;
      var p: number;
      r += 1;
      if (
        !isNaN(this.targetAntObj.hp) &&
        vis[this.targetAntObj.X][this.targetAntObj.Y] != -1
      )
        goal = this.targetAntObj;
      else
        for (p = 0; p < this.ants.length; p++) {
          if (
            vis[this.ants[p].X][this.ants[p].Y] != -1 &&
            vis[this.ants[p].X][this.ants[p].Y] < r &&
            this.ants[p].hp > 0
          ) {
            goal = this.ants[p];
            r = vis[this.ants[p].X][this.ants[p].Y];
          }
        }
      if (goal != nullAnt) {
        goal.hp -= damage;

        var Direction: number[];
        if (goal.Y % 2)
          Direction = [
            30 * goal.Y,
            20 * Math.sqrt(3) * goal.X + 10 * Math.sqrt(3),
          ];
        else Direction = [30 * goal.Y, 20 * Math.sqrt(3) * goal.X];

        var Direction2: number[];
        if (theTower.Y % 2)
          Direction2 = [
            30 * theTower.Y,
            20 * Math.sqrt(3) * theTower.X + 10 * Math.sqrt(3),
          ];
        else Direction2 = [30 * theTower.Y, 20 * Math.sqrt(3) * theTower.X];

        // 计算 点1指点2形成 的向量
        let a = [Direction[0] - Direction2[0], Direction[1] - Direction2[1]];
        let b = [0, -1];
        // 点积
        let dotProduct = a[0] * b[0] + a[1] * b[1];
        let d =
          Math.sqrt(a[0] * a[0] + a[1] * a[1]) *
          Math.sqrt(b[0] * b[0] + b[1] * b[1]);
        let angle = (Math.acos(dotProduct / d) * 180) / Math.PI;

        if (a[0] < 0) angle = 360 - angle;

        canvas
          .getObjects()
          .filter((element) => {
            return element.name == "T_" + theTower.id;
          })
          .forEach((element) => {
            // 旋转动画
            element.rotate(angle);
          });

        var dot = new fabric.Circle({
          left: Direction2[0] + 120,
          top: Direction2[1] + 30 + 10 * Math.sqrt(3),
          radius: 2 * theTower.level + 1,
          originX: "center",
          originY: "center",
          fill: "black",
          objectCaching: false,
        });
        canvas.add(dot);

        dot.animate("left", Direction[0] + 120, {
          duration: 500,
          onChange: canvas.renderAll.bind(canvas),
          easing: fabric.util.ease.easeInCubic,
        });
        dot.animate("top", 30 + Direction[1] + 10 * Math.sqrt(3), {
          duration: 500,
          onChange: canvas.renderAll.bind(canvas),
          easing: fabric.util.ease.easeInCubic,
          onComplete: function () {
            canvas.remove(dot);
          },
        });
      }
    },

    routePheromone(theAnt: Ant, val: number) {
      var p: number;
      for (p = 0; p < theAnt.path.length; p++) {
        this.pheromone[theAnt.path[p][0]][theAnt.path[p][1]][
          theAnt.path[p][2]
        ] += val / theAnt.path.length;
        if (
          this.pheromone[theAnt.path[p][0]][theAnt.path[p][1]][
            theAnt.path[p][2]
          ] < (this.minPhe as number)
        )
          this.pheromone[theAnt.path[p][0]][theAnt.path[p][1]][
            theAnt.path[p][2]
          ] = (this.minPhe as number) * 1;
      }
    },

    decayPheromon() {
      var X: number;
      var Y: number;
      // init canvas
      for (X = 0; X <= 20; X++)
        for (Y = 0; Y <= 20; Y++) {
          if (
            Y < 10 - (2 * X + 1) ||
            Y > 10 + (2 * X + 1) ||
            Y < 2 * X - 30 ||
            Y > 50 - 2 * X
          ) {
            continue;
          }
          var dir: number[][];
          if (Y % 2)
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
          var p: number;
          for (p = 0; p < 6; p++) {
            if (
              Y + dir[p][1] < 10 - (2 * (X + dir[p][0]) + 1) ||
              Y + dir[p][1] > 10 + (2 * (X + dir[p][0]) + 1) ||
              Y + dir[p][1] < 2 * (X + dir[p][0]) - 30 ||
              Y + dir[p][1] > 50 - 2 * (X + dir[p][0]) ||
              Y + dir[p][1] < 0 ||
              Y + dir[p][1] > 20
            )
              continue;
            this.pheromone[X][Y][p] *= this.decayRate as number;
            if (this.pheromone[X][Y][p] < (this.minPhe as number))
              this.pheromone[X][Y][p] = (this.minPhe as number) * 1;
          }
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
