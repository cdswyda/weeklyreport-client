<template>
  <Table ref="table" class="summary-table table-support-mobile" :columns="weeks > 1 ? column2 : column1" :data="data" :loading="loading" :class="{'multi-weeks': weeks > 1}">
    <template slot-scope="{ row }" slot="percent">
      <div :style="'color:' + rate2color(row.saturation * 100)">{{row.saturation | toPercent}}</div>
    </template>
    <template slot-scope="{ row }" slot="workList">
      <ListDisplay :data="row.workList" />
    </template>
    <template slot-scope="{ row }" slot="leaveList">
      <ListDisplay :data="row.leaveList" />
    </template>
    <template slot-scope="{ row }" slot="submitInfo">
      <span v-if="row.unSubmit" style="color:#ed4014">未提交</span>
      <span e-else>{{row.submitDate | dateTime}}</span>
    </template>
  </Table>
</template>

<script>
import ExpandRow from './ExpandRow';
import ListDisplay from './ListDisplay';

import rate2color from '@/util/rate2color.js';
import {date2text} from '@/util/date.js';

export default {
  name: 'summary-table',
  components: {
    // eslint-disable-next-line
    ExpandRow,
    ListDisplay
  },
  props: {
    weeks: {
      type: Number,
      default: 1
    },
    loading: {
      type: Boolean,
      default: false
    },
    data: {
      type: Array,
      default: () => []
    }
  },
  filters: {
    dateTime(v) {
      if (!v) return '';
      const d = new Date(v);
      return date2text(d, true);
    }
  },
  data() {
    return {
      column1: [
        { title: '姓名', key: 'name', sortable: true, width: 90 },
        { title: '工作内容', key: 'workList', slot: 'workList' },
        { title: '任务耗时', key: 'taskTime', sortable: true, width: 100 },
        { title: '沟通耗时', key: 'communicationTime', sortable: true, width: 100 },
        { title: '学习耗时', key: 'studyTime', sortable: true, width: 100 },
        { title: '饱和度', key: 'saturation', sortable: true, width: 100, slot: 'percent' },
        { title: '备注', key: 'leaveList', slot: 'leaveList', width: 200 },
        { title: '提交时间', key: 'submitDate', slot: 'submitInfo', width: 90 }
      ].map(c => {
        c.className = 'summary-column ' + (c.key || c.type);
        return c;
      }),
      column2: [
        {
          type: 'expand',
          slot: 'content',
          width: 50,
          render: (h, { row }) => {
            return h(ExpandRow, {
              props: { row }
            });
          }
        },
        { title: '姓名', key: 'name', sortable: true, width: 90 },
        { title: '任务耗时', key: 'taskTime', sortable: true, width: 100 },
        { title: '沟通耗时', key: 'communicationTime', sortable: true, width: 100 },
        { title: '学习耗时', key: 'studyTime', sortable: true, width: 100 },
        { title: '请假耗时', key: 'leaveTime', sortable: true, width: 100 },
        { title: '饱和度', key: 'saturation', sortable: true, width: 100, slot: 'percent' },
        { title: '时间范围', key: 'weekRange' }
      ].map(c => {
        c.className = 'summary-column ' + (c.key || c.type);
        return c;
      })
    };
  },
  methods: {
    rate2color,
    exportCsv(opt) {
      this.$refs.table.exportCsv(opt);
    }
  }
};
</script>

<style lang="scss">
.summary-table {
  // .summary-column.workList,
  // .summary-column.leaveList {
  //   display: none;
  // }
  // &.show-content {

  // }
  .ivu-table-expanded-cell {
    background: #fcfcfc;
  }
}
.summary-column .ivu-table-cell {
  padding-left: 8px;
  padding-right: 8px;
}
@media only screen and (max-width: 760px), (min-device-width: 768px) and (max-device-width: 1024px) {
  .table-support-mobile {
    td:last-child {
      border-bottom-width: 2px;
      // border-bottom-style: double;
    }
    &.summary-table {
      table {
        width: auto !important;
      }
      li {
        white-space: normal;
        margin-bottom: 4px;
      }
      td > * {
        padding-top: 6px;
      }
      td:nth-of-type(1):before {
        content: '姓名';
      }
      td:nth-of-type(2):before {
        content: '工作内容';
      }
      td:nth-of-type(3):before {
        content: '任务耗时';
      }
      td:nth-of-type(4):before {
        content: '沟通耗时';
      }
      td:nth-of-type(5):before {
        content: '学习耗时';
      }
      td:nth-of-type(6):before {
        content: '饱和度';
      }
      td:nth-of-type(7):before {
        content: '备注';
      }
      td:nth-of-type(8):before {
        content: '提交时间';
      }
      &.multi-weeks {
        td:nth-of-type(1) {
          // visibility: hidden;
          &:before {
            content: '查看详情';
          }
          .ivu-table-cell-with-expand {
            position: absolute;
            left: 0;
            right: 0;
            top: 0;
            bottom: 0;
            opacity: 0;
          }
        }
        td:nth-of-type(2):before {
          content: '姓名';
        }
        td:nth-of-type(3):before {
          content: '任务耗时';
        }
        td:nth-of-type(4):before {
          content: '沟通耗时';
        }
        td:nth-of-type(5):before {
          content: '学习耗时';
        }
        td:nth-of-type(6):before {
          content: '请假耗时';
        }
        td:nth-of-type(7):before {
          content: '饱和度';
        }
        td:nth-of-type(7):before {
          content: '时间范围';
        }
        td:nth-of-type(8):before {
          content: '备注';
        }
        td[colspan].ivu-table-expanded-cell {
          visibility: visible;
          &:before {
            content: '工作内容及请假情况';
          }
        }
      }
    }
  }
}
</style>