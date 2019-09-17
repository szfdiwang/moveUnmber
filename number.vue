<template>
  <div class="warp_box">
 
    <div class="cardLine">
      <div class="card_box" v-for="card in cardList" :key="card.id">
        <div class="fll">
          <img :src="card.img" />
        </div>
        <div class="fll rightContent">
          <span class="title">{{card.title}}</span>
          <p style="margin-top:10px">
            <span class="num" :ref="card.id"></span>
            <span>{{ card.id.includes('Rate') ? '%': '' }}</span>
          </p>

          <!-- <span class="num">{{card.number}}</span> -->
        </div>
      </div>
    </div>
   
  </div>
</template>
<script>
import * as utils from "@/utils/util.js";
import { CountUp } from "countup.js";
import { homeApi } from "@/api/index";
export default {
  name: "apiMgt",
  data() {
    return {
      countUp: [], // 保存数字动画特效
      showNoData: false,
      oneDayTimeStamp: 86400 * 1000,
      curSelectBtn: 0,
      servicetDate: "",
      isDisabled: true,
      dataReportTableData: [],
      dataReportDate: "",
      selectService: "" ,
      serviceList: [],
      selectApi: "",
      apiList: [],
      totalCall: "",
      taotalError: "",
      successRate: "",
      errorRate: "",
      totalNum: 0,
      routerId: "",
      serviceId: "",
      btnList: [
        { id: 0, label: "今天", value: 1 },
        { id: 1, label: "7天", value: 7 },
        { id: 2, label: "30天", value: 30 }
      ],
      cardList: [
        {
          id: "totalNum",
          title: "总调用数",
          img: require("../../assets/img/homepage/totalUse.png"),
          number: 0
        },
        {
          id: "errorNum",
          title: "总错误数",
          img: require("../../assets/img/homepage/error.png"),
          number: 0
        },
        {
          id: "successRate",
          title: "成功率",
          img: require("../../assets/img/homepage/success.png"),
          number: 0
        },
        {
          id: "failRate",
          title: "错误率",
          img: require("../../assets/img/homepage/failed.png"),
          number: 0
        }
      ]
    };
  },
  mounted() {
    let that = this;
    that.cardList.map(item => {
      const countUpItem = new CountUp(that.$refs[item.id][0], 0, {
        duration: 2
      });
      countUpItem.start();
      that.countUp.push(countUpItem);
    });
    that.initPage();
  },
  watch: {},
  methods: {
    //初始化页面
    initPage() {
      this.initSelectOneList();
      this.initOverviewInfo();
      this.queryEchartData();
      this.initTable();
    },
    //导出count报表
    exportCountFn() {
      let data = {
        routeId: this.selectApi,
        serviceId: this.selectService,
        startTime: this.dataReportDate[0] ? this.dataReportDate[0] : 0,
        endTime: this.dataReportDate[1] ? this.dataReportDate[1] : 0,
        type: 2
      };
      homeApi.queryCountListExcel(data).then(res => {
        if (res.data) {
          utils.exportFn(res.data, "数据报表.xlsx");
        }
      });
    },
    //导出排名报表
    exportRankFn() {
      let data = {
        serviceId: this.selectService,
        startTime: this.servicetDate[0] ? this.servicetDate[0] : "",
        endTime: this.servicetDate[1] ? this.servicetDate[1] : ""
      };
      homeApi.queryRankListExcel(data).then(res => {
        if (res.data) {
          utils.exportFn(res.data, "服务排行榜.xlsx");
        }
      });
    },
    //点击选择时间段触发事件
    handleDateChangeForDataReport(val) {
      //重置下面时间 拿掉样式
      this.curSelectBtn = -1;
      let data = {
        routeId: this.selectApi,
        serviceId: this.selectService,
        startTime: this.dataReportDate[0] ? this.dataReportDate[0] : 0,
        endTime: this.dataReportDate[1] ? this.dataReportDate[1] : 0,
        type: 2
      };
      homeApi.queryCountList(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.initReportChart(res.data.data);
        }
      });
    },
    //查询Echart数据
    queryEchartData(
      startTime = new Date(new Date().toLocaleDateString()).getTime(),
      endTime = Date.parse(new Date()),
      type = 1
    ) {
      let data = {
        routeId: this.selectApi,
        serviceId: this.selectService,
        startTime,
        endTime,
        type
      };
      homeApi.queryCountList(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.initReportChart(res.data.data);
        }
      });
    },
    //点击切换 1 7 30 天事件
    changeBtn(index) {
      this.dataReportDate = [];
      this.curSelectBtn = index;
      let str,
        type,
        oneday = new Date(new Date().toLocaleDateString()).getTime(),
        end = Date.parse(new Date());
      if (index === 0) {
        //小时
        type = 1;
        str = oneday;
      } else {
        //日期
        type = 2;
        if (index === 1) {
          //7天
          str = oneday - this.oneDayTimeStamp * 6;
        } else if (index === 2) {
          //30天
          str = oneday - this.oneDayTimeStamp * 29;
        }
      }
      this.queryEchartData(str, end, type);
    },
    //点击服务排行榜日期切换事件
    handleDateChange() {
      this.initTable();
    },
    //初始化表格方法
    initTable(currentPage = 1, pageSize = 10) {
      let data = {
        currentPage,
        pageSize,
        serviceId: this.selectService,
        startTime: this.servicetDate[0] ? this.servicetDate[0] : "",
        endTime: this.servicetDate[1] ? this.servicetDate[1] : ""
      };
      homeApi.queryRankList(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.dataReportTableData = res.data.data.list;
          this.totalNum = res.data.data.total;
        }
      });
    },
    //初始化卡片数据方法
    initOverviewInfo() {
      let that = this;
      let data = {
        serviceId: this.selectService,
        routeId: this.selectApi
      };
      homeApi.getOverviewInfo(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.cardList[0].number = res.data.data.totalNum;
          this.cardList[1].number = res.data.data.errorNum;
          this.cardList[2].number = res.data.data.successRate;
          this.cardList[3].number = res.data.data.failureRate;
        }
        this.updateUmber();
      });
    },
    updateUmber() {
      this.cardList.map((item, index) => {
        this.countUp[index].update(Number(item.number));
      });
    },
    //整体下拉1菜单切换方法
    handleChangeSelect(val) {
      if (!!val) {
        this.selectApi = "";
        this.isDisabled = false;
        this.initSelectApiList(val);
      } else {
        this.selectApi = "";
        this.isDisabled = true;
      }
      this.initPage();
    },
    //整体下拉2API菜单切换方法
    initSelectApiList(val) {
      this.serviceId = val;
      let data = {
        serviceId: val //
      };
      homeApi.getServiceAndApiInfo(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.apiList = res.data.data;
        }
      });
    },
    //下拉2动作切换
    handleApiSelect(val) {
      //routerId
      this.selectApi = val;
      this.initPage();
    },
    //初始化下拉1数据
    initSelectOneList() {
      let data = {};
      homeApi.getServiceAll(data).then(res => {
        if (res.data && res.data.code == 200) {
          this.serviceList = res.data.data;
        }
      });
    },
    //渲染Echart方法
    initReportChart(array) {
      let xData, yData1, yData2, yData3;
      if (array === undefined || array["totalNum"].length == 0) {
        this.showNoData = true;
        // return;
      } else {
        this.showNoData = false;
        xData = array["position"];
        yData1 = array["totalNum"];
        yData2 = array["errorNum"];
        yData3 = array["avgCost"];
      }
      let id = "dataReport_Echart";
      let option = {
        tooltip: {
          trigger: "axis"
        },
        xAxis: {
          type: "category",
          data: xData
        },
        yAxis: {
          type: "value"
        },
        legend: {
          x: "20",
          y: "12",
          icon: "circle",
          data: ["请求总数", "请求错误数", "平均耗时"]
        },
        series: [
          {
            name: "请求总数",
            data: yData1,
            type: "line"
          },
          {
            name: "请求错误数",
            data: yData2,
            type: "line"
          },
          {
            name: "平均耗时",
            data: yData3,
            type: "line"
          }
        ],
        grid: {
          x: 20,
          y: 50,
          x2: 20,
          y2: 20
        },
        color: ["#3F64E4", "#EB6D5D", "#05C97F"]
      };
      let thatChart = document.getElementById(id);
      let echart = this.$echarts.init(thatChart);
      echart.setOption(option);
      window.addEventListener("resize", function() {
        echart.resize();
      });
    }
  }
};
</script>
<style lang="scss" scoped>
.warp_box {
  padding: 20px;
  .selectLine {
    :nth-child(2) {
      margin-left: 10px;
    }
  }
  .cardLine {
    margin: 20px 0;
    height: auto;
    width: 100%;
    display: flex;
    flex-wrap: nowrap;
    justify-content: space-between;
    .card_box {
      width: 3.8rem;
      height: 108px;
      background: rgba(255, 255, 255, 1);
      border: 1px solid rgba(225, 227, 230, 1);
      padding: 25px 20px;
      .rightContent {
        margin-left: 20px;
        display: flex;
        flex-direction: column;
        .title {
          height: 20px;
          font-size: 14px;
          font-weight: 200;
          color: rgba(51, 51, 51, 1);
          line-height: 20px;
        }
        .num {
          margin-top: 10px;
          height: 24px;
          font-size: 26px;
          font-weight: 400;
          color: rgba(34, 34, 34, 1);
          line-height: 24px;
        }
      }
    }
  }
  .reportLine {
    height: auto;
    background: rgba(255, 255, 255, 1);
    padding: 20px;
    border: 1px solid rgba(232, 235, 238, 1);
    margin-bottom: 20px;
    position: relative;
    #dataReport_Echart {
      width: 100%;
      height: 3.68rem;
    }
    .title_line {
      height: 41px;
      border-bottom: 1px solid rgba(232, 235, 238, 1);
    }
    .tableLine {
      margin-top: 20px;
      height: auto;
      background: rgba(255, 255, 255, 1);
      border: 1px solid rgba(225, 227, 230, 1);
    }
  }
}
</style>
