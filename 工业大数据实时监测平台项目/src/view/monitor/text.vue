<template>
  <div>
    <Card :dis-hover="true" class="headerCard">
      <Select
        v-model="equipmentId"
        filterable
        placeholder="请选择机器"
        label-in-value
        @on-change="handleChooseMac"
      >
        <Option v-for="item in equipmentList" :value="item.id" :key="item.id">{{ item.name }}</Option>
      </Select>
    </Card>
    <Card :dis-hover="true" v-show="equipmentId" style="margin: 0 0px 20px 0px; ">
      <Row style="margin: -20px 0 0 -20px">
        <i-col
          :xs="12"
          :md="8"
          :lg="4"
          v-for="(infor, i) in inforCardData"
          :key="`infor-${i}`"
          style="height: 120px;padding: 10px;"
        >
          <infor-card
            shadow
            :color="infor.color"
            :icon="infor.icon"
            :icon-size="36"
            style="margin: 12px"
          >
            <count-to :end="infor.count" count-class="count-style"/>
            <p>{{ infor.title }}</p>
          </infor-card>
        </i-col>
      </Row>
    </Card>
    <Card :dis-hover="true">
      <hchartsLine :options="line" ref="line"></hchartsLine>
    </Card>
  </div>
</template>
<script>
import { hchartsLine } from "_c/charts";
import { wsConnect } from "@/api/index.js";
import { getMachines } from "@/api/equipment.js";
import { errMessage } from "@/api/index.js";
import CountTo from "_c/count-to";
import InforCard from "_c/info-card";
let list = new Map();
let times = 0;
export default {
  components: {
    hchartsLine,
    CountTo,
    InforCard
  },
  data() {
    return {
      dataArray:[],
      inforCardData: [],
      equipmentList: [],
      equipmentId: "",
      numC :1,

      line: {
        title: "实时参数监控", // 标题  没有则可去掉
        // subtitle: "实时参数监控",
        series: []
      },
      list: null
    };
  },
  methods: {
    async getMachines() {
      try {
        const machineList = await getMachines();
        for (let i = 0, length = machineList.data.msg.length; i < length; i++) {
          this.equipmentList.push({
            id: machineList.data.msg[i].machineId,
            name: machineList.data.msg[i].machineName
          });
        }
      } catch (err) {
        if (err.response !== undefined) {
          errMessage(err.response.status);
        } else {
          errMessage(1);
        }
      }
    },
    handleChooseMac() {
      this.numC =1
      this.clearLine();
      this.initWebSocket();
    },
    clearLine() {
      this.chart && this.chart.destroy();
      this.chart = null;
      list = null;
      list = new Map();
      this.line = {
        title: "",
        subtitle: "实时参数监控",
        series: []
      };
      this.inforCardData = [];
    },
    initWebSocket() {
      let msg = "1:" + this.equipmentId;
      this.socketApi.sendSock(msg, this.getSocketVal);
    },
    getSocketVal(res) {
      let that = this
      const temp = new Date().getTime();
      if (res.machineId === this.equipmentId) {
        if (list.get(res.physicalName) !== undefined) {
          if (list.get(res.physicalName).length > 40) {
            list.get(res.physicalName).shift();
          } else {
            list.get(res.physicalName).push([res.time, res.value]);
          }
        } else {
          list.set(res.physicalName, [[res.time, res.value]]);
        }
        console.log("===");
        console.log(list);
        let _i = 0;
        // that.dataArray = []
        if (temp - times > 1000) {
          times = temp;
          for (let [key, value] of list.entries()) {
            this.inforCardData.splice(_i, 1, {
              title: key,
              count: value[value.length - 1][1]
            });
            // for (let z =0 ;z<that.dataArray.length;z++)
            // {
            //   if(that.dataArray[z].name ===res.physicalName)
            //   {
            //     that.dataArray[z].data = value
            //   }
            // }
            // this.line.series.splice(_i, 1, {
            //   name: key,
            //   data: value
            // });
            _i++;
          }
        setTimeout(function () {
          if(that.numC === 1)
          {
            that.numC += 1;
            let _z = 0
            for (let [key, value] of list.entries()) {
              that.line.series.splice(_z, 1, {
                name: key,
                data: value
              });
              _z++;
            }
          }
          console.log(that.numC)
        },0)

        }
        // this.line = {
        //   title: "实时参数监控",
        //   series: this.dataArray
        // }
        _i = 0;
      } else if (typeof res !== Object) {
        if (res > 0) {
          this.$Message.error({
            duration: 1,
            content: "当前有异常数据！！！共" + res + "条"
          });
        }
        // this.$Message.warning("该机器无数据！");
      }
    }
  },
  created() {},
  beforeMount() {
    this.getMachines();
    if (this.$store.state.monitor.isGeneralChoose) {
      this.equipmentId = this.$store.state.monitor.machId;
      this.handleChooseMac();
    }
  },
  destroyed() {}
};
</script>
<style lang="less">
.count-style {
  font-size: 40px;
}
</style>

