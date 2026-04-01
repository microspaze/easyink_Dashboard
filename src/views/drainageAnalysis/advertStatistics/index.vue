<!--
 * @Description: 广告统计
 * @Author: system
 * @LastEditors: system
-->
<template>
  <div class="overview-page">
    <!-- 选择渠道弹窗 -->
    <SelectChannel
      :visible.sync="dialogVisibleSelectChannel"
      :confirm-selected-code-list="channelList"
      @success="selectedChannel"
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
            :col-list="colList"
          >
            <template slot="operate">
              <el-popover
                trigger="hover"
                placement="left-start"
              >
                <div class="popover-content">
                  <div class="info">
                    <p>总进线数：查询时间内的总进线数量</p>
                    <p>表单提交数：查询时间内提交表单的数量</p>
                    <p>表单提交率：表单提交数/总进线数</p>
                    <p>订单支付数：查询时间内支付订单的数量</p>
                    <p>订单支付率：订单支付数/总进线数</p>
                    <p>企微添加数：查询时间内添加企微的数量</p>
                    <p>企微添加率：企微添加数/总进线数</p>
                    <p>客户删除数：查询时间内删除客户的数量</p>
                    <p>客户删除率：客户删除数/总进线数</p>
                    <p>无效进线数：查询时间内总进线数-企微添加数</p>
                    <p>无效进线率：无效进线数/总进线数</p>
                  </div>
                </div>
                <div slot="reference" class="statistic theme-text-color">统计说明</div>
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
              <el-radio-button :label="CHANNEL_DIMENSION">渠道维度</el-radio-button>
            </el-radio-group>
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
            <el-table-column prop="totalCnt" label="总进线数" min-width="120" />
            <el-table-column prop="formedCnt" label="表单提交数" min-width="120" />
            <el-table-column prop="formedRate" label="表单提交率" min-width="120">
              <template #default="{ row }">
                {{ row.formedRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.formedRate + '%' }}
              </template>
            </el-table-column>
            <el-table-column prop="paidCnt" label="订单支付数" min-width="120" />
            <el-table-column prop="paidRate" label="订单支付率" min-width="120">
              <template #default="{ row }">
                {{ row.paidRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.paidRate + '%' }}
              </template>
            </el-table-column>
            <el-table-column prop="addedCnt" label="企微添加数" min-width="120" />
            <el-table-column prop="addedRate" label="企微添加率" min-width="120">
              <template #default="{ row }">
                {{ row.addedRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.addedRate + '%' }}
              </template>
            </el-table-column>
            <el-table-column prop="deletedCnt" label="客户删除数" min-width="120" />
            <el-table-column prop="deletedRate" label="客户删除率" min-width="120">
              <template #default="{ row }">
                {{ row.deletedRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.deletedRate + '%' }}
              </template>
            </el-table-column>
            <el-table-column prop="invalidCnt" label="无效进线数" min-width="120" />
            <el-table-column prop="invalidRate" label="无效进线率" min-width="120">
              <template #default="{ row }">
                {{ row.invalidRate === DATA_STATISTICS_DEFAULT_SHOW ? DATA_STATISTICS_DEFAULT_SHOW : row.invalidRate + '%' }}
              </template>
            </el-table-column>
          </el-table>
          <pagination
            :total="total * 1"
            :page.sync="pageQuery.pageNum"
            :limit.sync="pageQuery.pageSize"
            :disabled="loading"
            @pagination="getList()"
          />
        </div>
      </template>
    </RightContainer>
  </div>
</template>
<script>
import RightContainer from '@/components/RightContainer';
import Statistics from '@/components/Statistics.vue';
import { PAGE_LIMIT, DATA_STATISTICS_DEFAULT_SHOW } from '@/utils/constant/index';
import EmptyDefaultIcon from '@/components/EmptyDefaultIcon';
import SelectChannel from '../channelStatistics/components/SelectChannel.vue';
import { TODAY_TIME, FIXED_DAYS_AGO_TIME, ONE_MOUNTH_AGO, ONE_MOUNTH_LATER } from '@/utils/common';
import { getAdvertTotal, getAdvertChannelList } from '@/api/statistics';
import loadingMixin from '@/mixin/loadingMixin';

const CHANNEL_DIMENSION = 'channel';
const DATE_DIMENSION = 'date';
export default {
  name: 'AdvertStatistics',
  components: { RightContainer, Statistics, EmptyDefaultIcon, SelectChannel },
  mixins: [loadingMixin],
  data() {
    return {
      CHANNEL_DIMENSION,
      DATE_DIMENSION,
      DATA_STATISTICS_DEFAULT_SHOW,
      dimensionType: CHANNEL_DIMENSION,
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
        beginDate: undefined,
        endDate: undefined,
        empleCodeIdList: []
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
      // 表格loading
      loading: false,
      // 数据总览
      colList: [
        {
          totalCnt: 0,
          title: '总进线数',
          filed: 'totalCnt'
        },
        {
          formedCnt: 0,
          title: '表单提交数',
          filed: 'formedCnt'
        },
        {
          formedRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '表单提交率',
          unit: '%',
          filed: 'formedRate'
        },
        {
          paidCnt: 0,
          title: '订单支付数',
          filed: 'paidCnt'
        },
        {
          paidRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '订单支付率',
          unit: '%',
          filed: 'paidRate'
        },
        {
          addedCnt: 0,
          title: '企微添加数',
          filed: 'addedCnt'
        },
        {
          addedRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '企微添加率',
          unit: '%',
          filed: 'addedRate'
        },
        {
          deletedCnt: 0,
          title: '客户删除数',
          filed: 'deletedCnt'
        },
        {
          deletedRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '客户删除率',
          unit: '%',
          filed: 'deletedRate'
        },
        {
          invalidCnt: 0,
          title: '无效进线数',
          filed: 'invalidCnt'
        },
        {
          invalidRate: DATA_STATISTICS_DEFAULT_SHOW,
          title: '无效进线率',
          unit: '%',
          filed: 'invalidRate'
        }
      ],
      dialogVisibleSelectChannel: false,
      // 搜索框选择的渠道
      channelList: [],
      showChooseCodeTips: true
    };
  },
  methods: {
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
     * @description 选择渠道的回调
     */
    selectedChannel(list) {
      this.channelList = list;
    },
    // 获取数据总览
    getDataOverview() {
      getAdvertTotal(this.getSearchPayload()).then(res => {
        this.colList = this.colList.map((item) => {
          item[item.filed] = res?.data?.[item.filed];
          return item;
        });
      }).catch(() => {});
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
      getAdvertChannelList(this.getSearchPayload(), this.pageQuery).then((res) => {
        this.list = res.rows || [];
        this.total = res.total || 0;
      }).catch(() => {
        this.list = [];
      }).finally(() => {
        this.modifyButtonStatus();
        this.loading = false;
      });
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
      if (this.channelList && this.channelList.length) {
        this.query.empleCodeIdList = this.channelList.map((item) => item.id);
      } else {
        this.query.empleCodeIdList = [];
      }
      return this.query;
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
  .statistic {
    width: 60px;
    font-size: 14px;
    cursor: default;
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
</style>
