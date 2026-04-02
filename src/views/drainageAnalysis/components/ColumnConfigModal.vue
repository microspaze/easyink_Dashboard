<!--
 * @Description: 列配置弹窗
-->
<template>
  <el-dialog v-bind="$attrs" title="配置展示列" class="column-config-modal" v-on="$listeners" @close="onClose">
    <el-alert
      title="请勾选需要展示的数据列"
      type="warning"
      show-icon
      :closable="false"
    />
    <div class="check-list">
      <el-checkbox
        :value="isCheckAll"
        :indeterminate="isIndeterminate"
        @change="handleCheckAllChange"
      >全选</el-checkbox>
      <el-checkbox-group v-model="selected" class="checkbox-group">
        <el-checkbox
          v-for="item in list"
          :key="item.filed"
          :label="item.filed"
        >{{ item.title }}</el-checkbox>
      </el-checkbox-group>
    </div>
    <div slot="footer">
      <el-button @click="onClose">取消</el-button>
      <el-button type="primary" @click="handelConfirm">确定</el-button>
    </div>
  </el-dialog>
</template>
<script>
import store from '@/store';
export default {
  name: 'ColumnConfigModal',
  components: {},
  props: {
    list: {
      type: Array,
      default: () => []
    },
    // localStorage的key
    storageKey: {
      type: String,
      default: 'columnConfig'
    },
    // 是否全部选中
    checkAll: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      selected: [],
      isIndeterminate: false
    };
  },
  computed: {
    isCheckAll() {
      return this.selected.length === this.list.length && this.list.length > 0;
    }
  },
  watch: {
    selected(val) {
      this.isIndeterminate = val.length > 0 && val.length < this.list.length;
    }
  },
  mounted() {
    this.loadConfig();
  },
  methods: {
    /**
     * @description 加载已保存的配置
     */
    loadConfig() {
      const savedConfig = window.localStorage.getItem(`${this.storageKey}-${store.getters.corpId}`);
      if (savedConfig && savedConfig !== '') {
        try {
          const config = JSON.parse(savedConfig);
          this.selected = config.selected || [];
        } catch (e) {
          this.selected = [...this.list];
        }
      } else {
        this.selected = this.list.map(item => item.filed);
      }
    },
    /**
     * @description 全选/取消全选
     */
    handleCheckAllChange(val) {
      this.selected = val ? this.list.map(item => item.filed) : [];
    },
    /**
     * @description 关闭弹窗
     */
    onClose() {
      this.$emit('update:visible', false);
    },
    /**
     * @description 确认配置
     */
    handelConfirm() {
      if (!this.selected || this.selected.length === 0) {
        this.msgWarn('请至少选择一个数据列');
        return;
      }
      // 保存配置到localStorage
      window.localStorage.setItem(
        `${this.storageKey}-${store.getters.corpId}`,
        JSON.stringify({ selected: this.selected })
      );
      this.$emit('confirm', this.selected);
      this.onClose();
    }
  }
};
</script>
<style scoped lang='scss'>
.column-config-modal {
  .el-alert {
    margin-bottom: 15px;
  }
  .check-list {
    max-height: 300px;
    overflow: auto;
  }
  .checkbox-group {
    margin-top: 10px;
    display: flex;
    flex-direction: column;
  }
  ::v-deep .el-checkbox {
    margin-left: 0;
    margin-right: 20px;
    line-height: 32px;
  }
}
</style>
