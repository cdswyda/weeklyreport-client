<template>
  <div class="summary-report">
    <div class="summary-report-header">
      <div>
        <RangeSelect v-model="dateRange" class="mr-m" />
        <Button type="primary" :loading="loading" @click="getLog">查询</Button>
      </div>
      <span class="summary-report-title">{{summaryTitle}}</span>
      <div class="summary-report-action">
        <Select v-model="targetDept" v-if="deptList.length " style="text-align: left; width:120px" @on-change="getData">
          <Option v-for="item in deptList" :key="item.id" :value="item.id">{{item.name}}</Option>
        </Select>
        <Button :loading="exportLoading" @click="showFilterDialog = true">人员</Button>
        <Button type="primary" :loading="exportLoading" @click="exportData">导出</Button>
      </div>
    </div>
    <div ref="dataArea" class="summary-data-area">
      <h2>{{deptName}}</h2>
      <SummaryChart :loading="loading" @click="handleChartClick" :data="filterdList" style="margin-bottom:16px;" />
      <SummaryTable ref="summaryTable" :loading="loading" :data="filterdList" :weeks="weeks" />
    </div>

    <Modal title="请选择要显示的人员" v-model="showFilterDialog" okText="过滤" @on-ok="doFilter">
      <Tree :data="treeData" ref="tree" show-checkbox multiple></Tree>
    </Modal>
  </div>
</template>

<script>
import { getWeekStart, getWeekEnd, date2text, getDateByText } from '@/util/date.js';
import SummaryTable from '@/components/SummaryTable';
import SummaryChart from '@/components/SummaryChart';
import RangeSelect from '@/components/RangeSelect';
import scrollTo from '@/mixin/scrollto.js';
import exportData from '@/util/exportData.js';

const WEEK_MILLISECONDS = 1000 * 60 * 60 * 24 * 7;

const LOCAL_KEY = 'HIDDEN_USER_MAP';

export default {
  name: 'week-summary',
  components: { RangeSelect, SummaryTable, SummaryChart },
  mixins: [scrollTo],
  data() {
    return {
      loading: false,
      list: [],
      dateRange: {
        start: date2text(getWeekStart(new Date())),
        end: date2text(getWeekEnd(new Date()))
      },
      userList: [],
      exportLoading: false,
      summaryTitle: '',
      targetDept: '',
      deptList: [],
      hiddenUserIdMap: {},
      showFilterDialog: false
    };
  },
  computed: {
    deptName() {
      if (!this.targetDept) {
        return this.user.dept.name;
      }
      for (let i = 0; i < this.deptList.length; i++) {
        if (this.deptList[i].id == this.targetDept) {
          return this.deptList[i].name;
        }
      }
      return '';
    },
    user() {
      return this.$store.state.userData;
    },
    weeks() {
      const { start, end } = this.dateRange;
      let s = getDateByText(start);
      let e = getDateByText(end);
      e.setDate(e.getDate() + 1);
      return Math.ceil((e - s) / WEEK_MILLISECONDS) || 0;
    },
    weeksText() {
      if (this.weeks <= 1) return '';
      return `(共${this.weeks}周)`;
    },
    treeData() {
      if (!this.userList.length) {
        return [];
      }
      const hideUser = this.hiddenUserIdMap;
      // const hideUser = { '5de7884019186c688095d0b9': true };
      const groupMap = new Map();
      this.userList.forEach(u => {
        const gid = u.groupId;
        const itemData = groupMap.get(gid);
        if (!itemData) {
          groupMap.set(gid, {
            title: u.groupName,
            isGroup: true,
            expand: true,
            children: [
              {
                id: u.id,
                title: u.name,
                checked: hideUser[u.id] ? false : true
              }
            ]
          });
        } else {
          itemData.children.push({
            id: u.id,
            title: u.name,
            checked: hideUser[u.id] ? false : true
          });
        }
      });

      return [...groupMap.values()].map(group => {
        group.checked = group.children.every(item => item.checked);
        return group;
      });
    },
    filterdList() {
      if (!this.list.length) return [];
      const hide = this.hiddenUserIdMap;
      return this.list.filter(u => !hide[u.id]);
    }
  },
  mounted() {
    if (process.env.NODE_ENV != 'production') {
      this.dateRange.start = date2text(getWeekStart(new Date() - WEEK_MILLISECONDS));
      this.dateRange.end = date2text(getWeekEnd(new Date() - WEEK_MILLISECONDS));
    }
    if (this.user.dept && this.user.dept.id) {
      this.targetDept = this.user.dept.id;
      this.restoreLocal();
      if (this.user.role >= 100) {
        this.getDeptList();
      }
      this.getData();
    } else {
      // 用户无部门的情况下，取第一个部门来查询
      this.getDeptList().then(() => {
        const dept0 = this.deptList[0];
        if (dept0 && dept0.id) {
          this.targetDept = dept0.id;
          this.restoreLocal();
          this.getData();
        }
      });
    }
  },
  methods: {
    restoreLocal() {
      try {
        let localData = JSON.parse(localStorage.getItem(LOCAL_KEY + '-' + this.targetDept));
        if (localData && typeof localData == 'object') {
          this.hiddenUserIdMap = localData;
        }
      } catch (error) {
        console.error(error);
      }
    },
    getUserList() {
      this.loading = true;
      return this.$fetch(`api/user/list?dept=${this.targetDept || this.user.dept.id}`).then(res => {
        if (res.code == 200) {
          this.userList = res.data
            .map(u => {
              if (u.index === undefined) {
                u.index = 0;
              }
              if (u.dept && u.dept.leader == u.id) {
                u.index -= 100;
              } 
              if (u.group && u.group.leader == u.id) {
                u.index -= 10;
              }
              return u;
            })
            .sort((a, b) => {
              if (a.index === b.index) return 0;
              return a.index - b.index < 0 ? -1 : 1;
            })
            .sort((u1, u2) => {
              const a = u1.group.index;
              const b = u2.group.index;
              if (a === b) return 0;
              return a - b < 0 ? -1 : 1;
            })
            .map(u => {
              const extInfo = u.extInfo;
              return {
                id: u.id,
                name: u.name + (extInfo ? `(${extInfo})` : ''),
                groupId: u.group._id,
                groupName: u.group.name,
                deptId: u.dept._id,
                deptName: u.dept.name,
                reports: []
              };
            });
        } else {
          console.error(res.message);
        }
      });
    },
    getDeptList() {
      return new Promise(resolve => {
        if (this.user.role < 100) {
          this.deptList = [];
          resolve(this.deptList);
          return;
        }
        this.$fetch('api/dept/list')
          .then(res => {
            if (res.code == 200) {
              this.deptList = res.data;
            } else {
              this.deptList = [];
            }
            resolve(this.deptList);
          })
          .catch(err => {
            console.error(err);
            resolve([]);
          });
      });
    },
    getData() {
      this.getUserList().then(() => {
        this.getLog();
      });
    },
    getLog() {
      this.setSummaryTitle();
      this.loading = true;
      const cond = {
        ...this.dateRange,
        dept: this.targetDept || this.user.dept.id
      };
      this.$fetch('api/weeklyLog/weekRange', cond).then(res => {
        if (res.code === 200) {
          this.list = this.dealData(res.data);
        } else {
          this.$Message.error(res.message);
        }

        this.loading = false;
      });
    },
    dealData(data) {
      if (!Array.isArray(data)) return [];

      const userList = JSON.parse(JSON.stringify(this.userList));

      // 范围一周以上时的 多条记录的合并
      const logMap = {};
      data.forEach(item => {
        const uid = item.user.id || item.user._id;
        if (!uid) return;
        if (!logMap[uid]) {
          logMap[uid] = [item];
        } else {
          logMap[uid].push(item);
        }
      });

      userList.forEach(u => {
        const uid = u.id;
        const reports = logMap[uid];

        let summaryReport = {};
        // 未填
        if (!reports || !reports.length) {
          summaryReport = {
            week: '',
            weeks: 0,
            // start: item.startDate,
            // end: item.startEnd,
            taskTime: 0,
            communicationTime: 0,
            studyTime: 0,
            leaveTime: 0,
            workList: [],
            leaveList: [],

            saturationTime: 0,
            saturation: 0,
            standardTime: 40,
            unSubmit: true,
            submitDate: ''
          };
        } else {
          reports.sort((a, b) => {
            if (a.startDate == b.startDate) return 0;
            return +new Date(a.startDate) - +new Date(b.startDate) < 0 ? -1 : 1;
          });
          const report = reports[0].report;
          const { taskList, communicationList, studyList } = report;
          const workList = [...taskList, ...communicationList, ...studyList];
          const leaveList = [...report.leaveList];

          let taskTime = report.taskTime;
          let communicationTime = report.communicationTime;
          let studyTime = report.studyTime;
          let leaveTime = report.leaveTime;

          let saturationTime = taskTime + communicationTime;

          let startDate = reports[0].startDate;
          let endDate = reports[0].endDate;

          // todo 应从工作日中获取
          let standardTime = 40;

          reports.slice(1).forEach(it => {
            const r = it.report;
            workList.push(...r.taskList, ...r.communicationList, ...r.studyList);
            leaveList.push(...r.leaveList);

            taskTime += r.taskTime;
            communicationTime += r.communicationTime;
            studyTime += r.studyTime;
            leaveTime += r.leaveTime;

            saturationTime += r.taskTime + r.communicationTime;
            leaveTime += r.leaveTime;
            // todo 应从工作日中获取
            standardTime += 40;
            endDate = it.endDate;
          });
          const saturation = saturationTime / standardTime;
          summaryReport = {
            weeks: reports.length,
            workList,
            leaveList,

            taskTime,
            communicationTime,
            studyTime,
            leaveTime,

            saturationTime,
            standardTime,
            saturation,

            startDate,
            endDate,
            unSubmit: false,
            submitDate: reports[0].meta.updateAt
          };
        }

        Object.keys(summaryReport).forEach(k => {
          if (/Time$/.test(k)) {
            summaryReport[k] = parseInt(summaryReport[k].toFixed(0), 10);
          }
          u[k] = summaryReport[k];
        });
        if (u.startDate && u.endDate) {
          u.weekRange = date2text(new Date(u.startDate)) + '~' + date2text(new Date(u.endDate));

          if (this.weeks > 1 && u.weeks) {
            u.weekRange += `(共计${u.weeks}周)`;
          }
        }
      });
      console.log(userList);
      return userList;
    },
    setSummaryTitle() {
      this.summaryTitle = `${this.dateRange.start} ~ ${this.dateRange.end} ${this.weeksText}`;
    },
    doFilter() {
      const checkeds = this.$refs.tree.getCheckedNodes().filter(node => !node.isGroup);
      // 默认全部隐藏
      const hide = this.userList.reduce((t, u) => {
        t[u.id] = true;
        return t;
      }, {});
      // 再将要显示的去掉
      checkeds.forEach(node => {
        delete hide[node.id];
      });
      // save
      localStorage.setItem(LOCAL_KEY + '-' + this.targetDept, JSON.stringify(hide));
      this.hiddenUserIdMap = hide;

      this.showFilterDialog = false;
    },
    exportData() {
      // alert('即将到来');

      // this.$refs.summaryTable.exportCsv({
      //   filename: `${this.summaryTitle}.csv`,
      //   original: false
      // });
      exportData(this.$refs.dataArea, this.list, this.summaryTitle);
    },
    handleChartClick(name) {
      console.log(name);
      const nameEls = this.$refs.summaryTable.$el.querySelectorAll('.summary-column.name');
      const targetEl = [].slice.call(nameEls).filter(td => td.textContent.indexOf(name) != -1)[0];
      if (targetEl) {
        const targetRow = targetEl.closest('.ivu-table-row');
        this.scrollTo(targetRow);
        targetRow && targetRow.classList.add('highlight');
        setTimeout(() => {
          targetEl.classList.remove('highlight');
        }, 1000);
      }
    }
  }
};
</script>

<style lang="scss">
.summary-data-area {
  padding-top: 16px;
  > h2 {
    text-align: center;
  }
}
.summary-report-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}
.summary-report-action {
  // width: 140px;
  text-align: right;
  > * {
    margin-left: 8px;
  }
}
.ivu-table-row.highlight {
  td {
    background: #ebf7ff;
  }
}
</style>