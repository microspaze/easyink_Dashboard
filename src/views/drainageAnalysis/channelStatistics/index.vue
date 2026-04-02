<!--
 * @Description: 渠道统计
 * @Author: wJiaaa
 * @LastEditors: wJiaaa
-->
<template>
  <div class="overview-page">
    <!-- 选择员工/部门弹窗 -->
    <SelectUser
      :visible.sync="dialogVisibleSelectUser"
      title="选择员工/部门"
      :is-only-leaf="false"
      is-dep-linkage
      :selected-user-list="userAndDepartmentList"
      @success="selectedUserOrDepartment"
    />
    <!-- 选择渠道弹窗 -->
    <SelectChannel
      :visible.sync="dialogVisibleSelectChannel"
      :confirm-selected-code-list="channelList"
      @success="selectedChannel"
    />
    <!-- 列配置弹窗 -->
    <ColumnConfigModal
      :visible.sync="dialogVisibleColumnConfig"
      :list="columnConfigList"
      storage-key="channelStatisticsColumnConfig"
      @confirm="handleColumnConfigConfirm"
    />
    <RightContainer>
      <template v-slot:search>
        <el-form
          ref="queryForm"
          :inline="true"
          :model="query"
          label-width="100px"
          class="top-search"
          size="small"
        >
          <el-form-item prop="code">
            <div class="tag-input" @click="dialogVisibleSelectChannel = true">
              <span v-if="!channelList.length" class="tag-place">请选择渠道</span>
              <template v-else>
                <div class="code-label">
                  {{ getCodeShowLabel(channelList) }}
                </div>
              </template>
            </div>
          </el-form-item>
          <el-form-item prop="name">
            <div class="tag-input" @click="dialogVisibleSelectUser = true">
              <span v-if="!userAndDepartmentList.length" class="tag-place">请选择员工/部门</span>
              <template v-else>
                <el-tag
                  v-for="(unit, unique) in userAndDepartmentList"
                  :key="unique"
                  class="theme-text-color user-tag iaic"
                >
                  <TagUserShow :name="unit.name" :show-icon="!unit.userId" />
                </el-tag>
              </template>
            </div>
          </el-form-item>
          <el-form-item>
            <el-date-picker
              v-model="dateRange"
              value-format="yyyy-MM-dd"
              type="daterange"
              range-separator="-"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
              :picker-options="pickerOptions"
              :clearable="false"
            />
          </el-form-item>
          <el-form-item>
            <el-button
              v-preventReClick="200"
              type="primary"
              :loading="searchButtonLoading"
              @click="()=>{
                searchButtonLoading = true;
                onSearch()
              }"
            >查询</el-button>
            <el-button
              class="btn-reset"
              @click="resetForm"
            >重置</el-button>
          </el-form-item>
        </el-form>
      </template>
      <template v-slot:data>
        <div class="data-overview">
          <Statistics
            title="数据总览"
            :col-list="filteredColList"
          >
            <template slot="operate">
              <el-popover
                trigger="hover"
                placement="left-start"
              >
                <div class="popover-content">
                  <div class="info">
                    <p>累计添加客户：截止到查询时间，累计通过渠道添加员工的客户数</p>
                    <p>留存客户总数：截止到查询时间，未将员工删除的客户数</p>
                    <p>重复客户数：查询时间内，重复添加客户数</p>
                    <p>新增客户数：查询时间内，通过渠道添加员工的客户数，包含流失客户</p>
                    <p>新客流失数：查询时间内，新增客户把员工删除的客户数</p>
                    <p>流失客户数：查询时间内，把员工删除的客户数</p>
                    <p>24小时流失数：查询时间内，24小时内把员工删除的客户总数</p>
                    <p>48小时流失数：查询时间内，48小时内把员工删除的客户总数</p>
                    <p>新客留存率：截止当前，未将员工删除的新增客户/新增客户数</p>
                  </div>
                </div>
                <div slot="reference" class="statistic theme-text-color" @click="dialogVisibleColumnConfig = true">统计说明</div>
              </el-popover>
            </template>
          </Statistics>
        </div>
        <div class="table-overview">
          <div class="detail-title">
            数据详情
          </div>
          <div class="forms-handle-btn">
            <el-radio-group v-model="dimensionType" size="medium" @input="dimensionTypeChange">
              <el-radio-button :label="CODE_DIMENSION">渠道维度</el-radio-button>
              <el-radio-button :label="STAFF_DIMENSION">员工维度</el-radio-button>
              <el-radio-button :label="DATE_DIMENSION">日期维度</el-radio-button>
            </el-radio-group>
            <el-button
              v-hasPermi="['stastistic:channelStatistics:export']"
              class="btn-reset btn-export"
              @click="exportForms"
            >导出报表</el-button>
          </div>
          <el-table
            ref="showTable"
            v-loading="loading"
            :data="list"
          >
            <template slot="empty">
              <empty-default-icon :length="list.length" :text="showChooseCodeTips ? '请先选择渠道' : '暂无数据'" />
            </template>
            <el-table-column
              v-if="dimensionType === STAFF_DIMENSION && visibleColumns.includes('userName')"
              prop="userName"
              label="员工"
              min-width="200"
            >
              <template #default="{ row }">
                <UserItem :data="row" :is-staff="true" />
              </template>
            </el-table-column>
            <el-table-column
              v-if="dimensionType === DATE_DIMENSION && visibleColumns.includes('time')"
              prop="time"
              label="日期"
              min-width="200"
            >
              <template #default="{ row }">
                {{ row.time }}
              </template>
            </el-table-column>
            <el-table-column
              v-if="dimensionType === CODE_DIMENSION && visibleColumns.includes('empleName')"
              prop="empleName"
              label="渠道"
              min-width="200"
            >
              <template #default="{ row }">
                <el-tooltip effect="dark" :content="row.empleName" placement="top">
                  <span class="intwoline">
                    {{ row.empleName }}
                  </span>
                </el-tooltip>
              </template>
            </el-table-column>
            <el-table-column
              v-if="visibleColumns.includes('accumulateCustomerCnt')"
              prop="accumulateCustomerCnt"
              label="累计添加客户"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('retainCustomerCnt')"
              prop="retainCustomerCnt"
              label="留存客户总数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('newCustomerCnt')"
              prop="newCustomerCnt"
              label="新增客户数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('lossNewCustomerCnt')"
              prop="lossNewCustomerCnt"
              label="新客流失数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('lossCustomerCnt')"
              prop="lossCustomerCnt"
              label="流失客户数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('loss24hCustomerCnt')"
              prop="loss24hCustomerCnt"
              label="24小时流失数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('loss48hCustomerCnt')"
              prop="loss48hCustomerCnt"
              label="48小时流失数"
              min-width="180"
            />
            <el-table-column
              v-if="visibleColumns.includes('retainNewCustomerRate')"
              prop="retainNewCustomerRate"
              label="新客留存率"
              min-width="180"
            >
              <template #default="{ row }">
                {{ row.retainNewCustomerRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.retainNewCustomerRate + '%' }}
              </template>
            </el-table-column>
          </el-table>
          <pagination
            :total="total * 1"
            :page.sync="pageQuery.pageNum"
            :limit.sync="pageQuery.pageSize"
            :disabled="loading"
            @pagination="handleChangePage()"
          />
        </div>
      </template>
    </RightContainer>
  </div>
</template>
<script>
import RightContainer from '@/components/RightContainer';
import Statistics from '@/components/Statistics.vue';
import { PAGE_LIMIT, DATE_DIMENSION, STAFF_DIMENSION, CODE_DIMENSION, DATA_STATISTICS_DEFAULT_SHOW } from '@/utils/constant/index';
import EmptyDefaultIcon from '@/components/EmptyDefaultIcon';
import UserItem from '@/components/UserItem.vue';
import TagUserShow from '@/components/TagUserShow';
import SelectChannel from './components/SelectChannel.vue';
import SelectUser from '@/components/SelectUser/index.vue';
import ColumnConfigModal from '../components/ColumnConfigModal.vue';
import { TODAY_TIME, FIXED_DAYS_AGO_TIME, ONE_MOUNTH_AGO, ONE_MOUNTH_LATER, groupByScopeType } from '@/utils/common';
import {
  exportStatisticsByDate,
  exportStatisticsByStaff,
  exportStatisticsByCode,
  listEmpleTotal,
  emplecodeByDate,
  emplecodeByStaff,
  emplecodeByCode
} from '@/api/statistics';
import loadingMixin from '@/mixin/loadingMixin';
import store from '@/store';
export default {
  name: '',
  components: { RightContainer, Statistics, EmptyDefaultIcon, UserItem, TagUserShow, SelectUser, SelectChannel, ColumnConfigModal },
  mixins: [loadingMixin],
  data() {
    return {
      DATE_DIMENSION,
      STAFF_DIMENSION,
      CODE_DIMENSION,
      DATA_STATISTICS_DEFAULT_SHOW,
      // 选择添加人弹窗显隐
      dialogVisibleSelectUser: false,
      // 搜索框选择的员工/部门
      userAndDepartmentList: [],
      // 日期选择器第一次选中的时间
      pickerMinDate: '',
      // 日期范围选择限定
      pickerOptions: {
        onPick: obj => {
          this.pickerMinDate = new Date(obj.minDate).getTime();
        },
        disabledDate: v => {
          if (this.pickerMinDate) {
            const minTime = new Date(ONE_MOUNTH_AGO(this.pickerMinDate)).getTime();
            const maxTime = new Date(ONE_MOUNTH_LATER(this.pickerMinDate)).getTime();
            return v.getTime() > maxTime || v.getTime() < minTime || v.getTime() < new Date(FIXED_DAYS_AGO_TIME).getTime() || v.getTime() > new Date(TODAY_TIME).getTime();
          } else {
            return v.getTime() < new Date(FIXED_DAYS_AGO_TIME).getTime() || v.getTime() > new Date(TODAY_TIME).getTime();
          }
        }
      },
      // 日期范围
      dateRange: [TODAY_TIME, TODAY_TIME],
      // 查询参数
      query: {
        departmentIds: [], // 部门id列表
        userIds: [], // 用户id列表
        beginDate: undefined,
        endDate: undefined
      },
      // 翻页参数
      pageQuery: {
        pageNum: 1,
        pageSize: PAGE_LIMIT
      },
      // 总条数
      total: 0,
      // 表格展示的列表数据
      list: [],
      // 日期维度查询出的列表所有页数据
      allListByDate: [],
      // 表格loading
      loading: false,
      // 数据总览
      colList: [
        {
          accumulateCustomerCnt: 0,
          title: '累计添加客户',
          filed: 'accumulateCustomerCnt'
        },
        {
          retainCustomerCnt: 0,
          title: '留存客户总数',
          filed: 'retainCustomerCnt'
        },
        {
          duplicateCustomerCnt: 0,
          title: '重复客户数',
          filed: 'duplicateCustomerCnt'
        },
        {
          newCustomerCnt: 0,
          title: '新增客户数',
          filed: 'newCustomerCnt'
        },
        {
          lossNewCustomerCnt: 0,
          title: '新客流失数',
          filed: 'lossNewCustomerCnt'
        },
        {
          lossCustomerCnt: 0,
          title: '流失客户数',
          filed: 'lossCustomerCnt'
        },
        {
          loss24hCustomerCnt: 0,
          title: '24小时流失数',
          filed: 'loss24hCustomerCnt'
        },
        {
          loss48hCustomerCnt: 0,
          title: '48小时流失数',
          filed: 'loss48hCustomerCnt'
        },
        {
          retainNewCustomerRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '新客留存率',
          unit: '%',
          filed: 'retainNewCustomerRate'
        }
      ],
      // 列配置列表
      columnConfigList: [
        { title: '累计添加客户', filed: 'accumulateCustomerCnt' },
        { title: '留存客户总数', filed: 'retainCustomerCnt' },
        { title: '重复客户数', filed: 'duplicateCustomerCnt' },
        { title: '新增客户数', filed: 'newCustomerCnt' },
        { title: '新客流失数', filed: 'lossNewCustomerCnt' },
        { title: '流失客户数', filed: 'lossCustomerCnt' },
        { title: '24小时流失数', filed: 'loss24hCustomerCnt' },
        { title: '48小时流失数', filed: 'loss48hCustomerCnt' },
        { title: '新客留存率', filed: 'retainNewCustomerRate' }
      ],
      // 当前可见的列
      visibleColumns: [],
      dimensionType: CODE_DIMENSION,
      dialogVisibleSelectChannel: false,
      dialogVisibleColumnConfig: false,
      // 搜索框选择的渠道
      channelList: [],
      showChooseCodeTips: true
    };
  },
  computed: {
    // 根据visibleColumns过滤后的总览数据
    filteredColList() {
      return this.colList.filter(item => this.visibleColumns.includes(item.filed));
    }
  },
  mounted() {
    this.initVisibleColumns();
  },
  methods: {
    /**
     * @description 初始化可见列
     */
    initVisibleColumns() {
      // 从localStorage加载保存的配置
      const savedConfig = window.localStorage.getItem(`channelStatisticsColumnConfig-${store.getters.corpId}`);
      if (savedConfig && savedConfig !== '') {
        try {
          const config = JSON.parse(savedConfig);
          this.visibleColumns = config.selected || this.columnConfigList.map(item => item.filed);
        } catch (e) {
          // 默认全部显示
          this.visibleColumns = this.columnConfigList.map(item => item.filed);
        }
      } else {
        // 默认全部显示
        this.visibleColumns = this.columnConfigList.map(item => item.filed);
      }
    },
    /**
     * @description 列配置确认
     */
    handleColumnConfigConfirm(selectedFields) {
      this.visibleColumns = selectedFields;
    },
    /**
     * 切换维度查询
     */
    dimensionTypeChange() {
      this.getList(true);
    },
    /**
     * 获取渠道展示的label
     * @param list 已经选择的渠道
     */
    getCodeShowLabel(list) {
      return list.map((item) => item.scenario).join('、');
    },
    /**
     * @description 选择员工/部门的回调
     */
    selectedUserOrDepartment(list) {
      this.userAndDepartmentList = list;
    },
    /**
     * @description 选择渠道的回调
     */
    selectedChannel(list) {
      this.channelList = list;
    },
    // 获取数据总览
    getDataOverview() {
      listEmpleTotal(this.getSearchPayload()).then(res => {
        this.colList = this.colList.map((item) => {
          item[item.filed] = res?.data?.[item.filed];
          return item;
        });
      }).catch(() => {});
    },
    /**
     * 获取接口分页参数
     */
    getSearchPageParams() {
      // 日期维度由前端分页
      if (this.dimensionType !== DATE_DIMENSION) {
        return this.pageQuery;
      }
      return {};
    },
    /**
     * @description 获取表格数据
     * @param {*} initPage  是否重置为第一页
     * @return {*}
     */
    getList(initPage) {
      if (!this.channelList.length) return;
      initPage ? this.pageQuery.pageNum = 1 : null;
      this.loading = true;
      const getListFnMap = {
        [DATE_DIMENSION]: emplecodeByDate,
        [STAFF_DIMENSION]: emplecodeByStaff,
        [CODE_DIMENSION]: emplecodeByCode
      };
      getListFnMap[this.dimensionType](this.getSearchPayload(), this.getSearchPageParams()).then((res) => {
        const resList = [...res.rows];
        if (this.dimensionType === DATE_DIMENSION) {
          this.allListByDate = resList;
          this.list = this.dealPaging(resList);
        } else {
          this.list = resList;
        }
        this.total = res.total || 0;
      }).catch(() => {
        this.list = [];
      }).finally(() => {
        this.modifyButtonStatus();
        this.loading = false;
      });
    },
    /**
     * @description 处理日期维度表格数据分页
     * @param list
     */
    dealPaging(list) {
      const { pageNum, pageSize } = this.pageQuery;
      return list.slice((pageNum - 1) * pageSize, pageNum * pageSize);
    },
    // 查询
    onSearch() {
      if (!this.channelList.length) {
        this.modifyButtonStatus();
        return this.msgWarn('请先选择渠道');
      }
      this.showChooseCodeTips = false;
      this.getDataOverview();
      this.getList(true);
    },
    // 重置
    resetForm() {
      this.userAndDepartmentList = [];
      this.query = this.$options.data().query;
      this.channelList = [];
      this.colList = this.$options.data().colList;
      this.list = [];
      this.pickerMinDate = '';
      this.showChooseCodeTips = true;
      this.dateRange = [TODAY_TIME, TODAY_TIME];
      this.pageQuery = this.$options.data().pageQuery;
      this.total = this.$options.data().total;
    },
    /**
     * @description 获取搜索的传参
     */
    getSearchPayload() {
      if (this.dateRange) {
        this.query.beginDate = this.dateRange[0];
        this.query.endDate = this.dateRange[1];
      } else {
        this.query.beginDate = '';
        this.query.endDate = '';
      }
      if (this.userAndDepartmentList && this.userAndDepartmentList.length > 0) {
        const allListObj = groupByScopeType(this.userAndDepartmentList);
        this.query.departmentIds = allListObj.useDepartmentList.map(item => item.id);
        this.query.userIds = allListObj.useEmployeesList.map(item => item.userId);
      } else {
        this.query.departmentIds = [];
        this.query.userIds = [];
      }
      if (this.channelList && this.channelList.length) {
        this.query.empleCodeIdList = this.channelList.map((item) => item.id);
      } else {
        this.query.empleCodeIdList = [];
      }
      return this.query;
    },
    // 导出报表
    exportForms() {
      if (!this.channelList.length) {
        return this.msgWarn('请先选择渠道');
      }
      // 根据配置的显示列生成导出字段
      const exportFieldsMap = {
        accumulateCustomerCnt: '累计添加客户',
        retainCustomerCnt: '留存客户总数',
        duplicateCustomerCnt: '重复客户数',
        newCustomerCnt: '新增客户数',
        lossNewCustomerCnt: '新客流失数',
        lossCustomerCnt: '流失客户数',
        loss24hCustomerCnt: '24小时流失数',
        loss48hCustomerCnt: '48小时流失数',
        retainNewCustomerRate: '新客留存率'
      };
      // 根据维度添加额外的导出字段
      if (this.dimensionType === CODE_DIMENSION) {
        exportFieldsMap.empleName = '渠道';
      } else if (this.dimensionType === STAFF_DIMENSION) {
        exportFieldsMap.userName = '员工';
      } else if (this.dimensionType === DATE_DIMENSION) {
        exportFieldsMap.time = '日期';
      }
      // 只导出显示的列
      const selectedFields = this.visibleColumns.map(field => exportFieldsMap[field]).filter(Boolean);
      if (selectedFields.length === 0) {
        return this.msgWarn('请至少选择一个导出列');
      }
      const payload = {
        ...this.getSearchPayload(),
        selectedFields
      };
      const exportFnMap = {
        [DATE_DIMENSION]: exportStatisticsByDate,
        [STAFF_DIMENSION]: exportStatisticsByStaff,
        [CODE_DIMENSION]: exportStatisticsByCode
      };
      exportFnMap[this.dimensionType](payload).then((res) => {
        this.download(res.data.msg, true);
      }).catch(() => {
        this.msgError('导出失败!');
      });
    },
    /**
     * 表格翻页
     */
    handleChangePage() {
      // 日期维度由前端分页，翻页时不需要重新获取数据
      if (this.dimensionType === DATE_DIMENSION) {
        this.list = this.dealPaging(this.allListByDate);
      } else {
        this.getList();
      }
    }
  }
};
</script>
<style scoped lang='scss'>
  .data-overview {
    height: 182px;
    width: 100%;
    background-color: #fff;
    margin-bottom: 10px;
    cursor: pointer;
  }
  .table-overview {
    padding: 15px;
    background-color: #fff;
    flex: 1;
     .detail-title {
      font-size: 24px;
      font-weight: bold;
      color: #000000;
      margin-bottom: 15px;
    }
    .forms-handle-btn {
      display: flex;
      justify-content: space-between;
      margin-bottom: 15px;
    }
  }
  ::v-deep .show-data-wrapper {
    background-color: transparent;
    padding: 0;
    display: flex;
    .data-container {
      width: 100%;
      display: flex;
      flex-direction: column;
    }
  }
  .popover-content {
    width: 390px;
    font-size: 12px;
    line-height: 20px;
    .line {
      margin: 10px 0 10px 0;
      width: 100%;
      height: 0;
      border-bottom: 1px solid #EEE;
    }
    .notice {
      color: #E29836;
    }
  }
  .tag-input {
    .code-label {
      height: 32px;
      color: #606266;
    }
  }
  .statistic {
    font-size: 14px;
    cursor: default;
  }
</style>
