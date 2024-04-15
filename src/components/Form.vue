<template>
  <el-form ref="form" class="form" label-position="top">
    
    <el-alert style="margin: 20px 0;background-color: #e1eaff;color: #606266;"  :title="$t('alerts.selectNumberField')" type="info" />

    <el-form-item :label="$t('labels.number')" size="large" required>
      <el-select v-model="numberFieldId" :placeholder="$t('placeholder.number')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-button type="primary" plain size="large">新增一行记录</el-button>
  </el-form>
</template>

<script setup>
import { bitable } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted } from 'vue';
import axios from 'axios';



// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([])
const interfaceAddr = ref('https://wuliu.market.alicloudapi.com/kdi')
const numberFieldId = ref('')
const appcode = ref('db7bd581534a4da2b13f60713ff83396')

// -- 核心算法区域
const getLocsInfo = async () => {
  const url = `${interfaceAddr.value}?no=${IsbnValue.value}`;

  const config = {
    headers: {
      'Authorization': `APPCODE ${appcode.value}`,
      'Content-Type': 'application/json; charset=UTF-8'
    }
  };

  let res = ''
  await axios.get(url, config)
    .then((response) => {
      isDataWriten.value = 1
      res = response
    }).catch(err => {
      // isDataWriten.value = 2
      // requestErrorInfo.value = err
    })

  console.log("res", res)
  // if (res.status === 200)
  //   return res.data.result
  // else
  //   return "ERROR"
};



// -- 辅助算法区域


onMounted(async () => {
  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldListSeView.value = await view.getFieldMetaList()
  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  // if (localStorage.getItem('ossConfig') !== null) {
  //   ossConfig.value = JSON.parse(localStorage.getItem('ossConfig')) 
  // }
  // if (localStorage.getItem('attchImgFieldId') !== null) {
  //   attchImgFieldId.value = localStorage.getItem('attchImgFieldId')
  // }
  // if (localStorage.getItem('linkFieldId') !== null) {
  //   linkFieldId.value = localStorage.getItem('linkFieldId')
  // }
  // 从缓存中获取数据 -- end
});
    
</script>



<style scoped>
  
</style>
