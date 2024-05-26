<template>
  <!-- TITLE  -->
  <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{ $t('title') }}</div>
  <!-- DESCRIPTION  -->
  <el-alert  style="margin: 20px 0 0 0;color: #606266;" :title="$t('alerts.selectNumberField')" type="info" />
  <el-alert  style="margin: 10px 0 0 0;" :title="$t('alerts.warning')" type="warning" />
  <!-- DOCX -->
  <el-link style="color: #3e75f5;margin: 20px 0;" type="primary" :href="DESC_DOCX_URL"
  target="_blank">👉  {{ $t('labels.apiDocument') }}</el-link>

  
  <el-form style="color: #606266">
    <!-- INPUT-AREA  -->
    <el-form-item :label="$t('labels.headers')  " size="large" required>
      Cookie<el-input v-model="IG_COOKIE" :placeholder="$t('placeholder.cookie')" />
      X-Ig-App-ID<el-input v-model="IG_APP_ID" :placeholder="$t('placeholder.appId')" />
      X-Ig-Www-Claim<el-input v-model="IG_CLAIM" :placeholder="$t('placeholder.claim')" />

    </el-form-item>

    <el-form-item :label="$t('labels.params') " size="large" required>
      Username<el-input v-model="targetUsername" :placeholder="$t('placeholder.username')" />
      Hashtag<el-input v-model="targetHashTag" :placeholder="$t('placeholder.hashtag')" />
    </el-form-item>


    <!-- CHECKBOX-AREA  -->
    <div class="map-fields-checklist">
      <el-checkbox v-model="selectAllFields" :indeterminate="isIndeterminateToMap" @change="handleselectAllFieldsChange">{{
        $t('selectGroup.selectAll') }}</el-checkbox>
      <el-checkbox-group v-model="responseFieldsSelected" @change="handleresponseFieldsSelectedChange">
        <el-checkbox v-for="fieldToMap in responseFieldsAvaiable" :key="fieldToMap.label" :label="fieldToMap.label">
          {{ $t(`selectGroup.${fieldToMap.label}`) }}
        </el-checkbox>
      </el-checkbox-group>
    </div>

    <!-- SUMBIT BUTTON  -->
    <el-button color="#3370ff" style="margin: 20px 0;;"  type="primary" @click="handleIGRequest">{{ $t('sumbit')
    }}</el-button>    
  </el-form>

  <!-- DESCRIPTION  -->
  <el-alert v-if="processResultDesc.length" style="background-color: #e1eaff;color: #606266;" :title="processResultDesc"  type="success" show-icon />
  


</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

// -- 配置区域
const { t } = useI18n();  // 国际化
const IG_HEADER_STRUCTURE = {
  "cookie": "cookie",
  "app_id": "app_id",
  "claim": "claim"
}

const IG_PARAMS_STRUCTURE = {
  "hashtag": "hashtag",
  "username": "user"
}

const DESC_DOCX_URL = "https://jfsq6znqku.feishu.cn/wiki/VHeGwRp37ixa9bk1LqdcVfOenyh?from=from_copylink"
const BASE_REQUEST_URL = "https://get-ig-data-by-link-wuyi.replit.app"

const BASE_I18N_FIELD_PATH = "selectGroup"
const BASE_I18N_FIELD_TYPE_CHECK_PATH = "fieldTypeChecks"

const TEXT_FIELD_ARRAY = ['text', 'postLink', 'videoLink']
const DATETIME_FIELD_ARRAY = ['createTime', 'requestTime']
const NUMBER_FIELD_ARRAY = ['videoViewCount', 'likeCount', 'commentCount']

// -- 核心数据区域
// --== 响应式
const IG_COOKIE = ref("")
const IG_APP_ID = ref("")
const IG_CLAIM = ref("")

const targetHashTag = ref("")
const targetUsername = ref("")

const responseFieldsSelected = ref([
  "postLink", "videoViewCount", "likeCount", "commentCount", "createTime", "requestTime"
]) //  可以创建的字段


// --== 非响应式


// -- 辅助数据区域
const processResultDesc = ref("")
const selectAllFields = ref(false)
const isIndeterminateToMap = ref(true)
const responseFieldsAvaiable = ref([
  {"label": "postLink"},
  {"label": "text"},
  {"label": "videoViewCount"},
  {"label": "likeCount"},
  {"label": "commentCount"},
  {"label": "videoLink"},
  {"label": "createTime"},
  {"label": "requestTime"}
  
])   
let postTotalNum = 0
let postNumFilteredHashtag = 0


// -- 方法声明：函数式 & 查询式

/**
 * @main {插件运行主函数}
 */
const handleIGRequest = async () => {
  // 检查必选信息是否已填写
  const checkRes = await checkIsRequiredInfoFilled()
  if (checkRes.isError)
    return

  // 插件运行开始提示
  await showProcessTip("start")

  // 查询 Base SDK table、view 等实例，并获取当前表格的字段元信息
  const {table, view, existedFieldMetaList} = await queryBaseTableAndView()
  

  // 查询 Instagram Cookie 等请求头信息
  const headers = queryIGHeaderInput() 

  // 查询 Instagram Hashtag 等请求参数信息
  const params = queryIGParamsInput()

  // 查询 Instagram response fields selected
  const targetResponseFields = queryTargetResponseFields()

  // 查询 targetResponseFields 对应的多维表格字段Id
  const baseTableFieldsIdTargeted = await queryBaseTableFieldsIdTargeted(table, existedFieldMetaList, targetResponseFields)
  console.log("baseTableFieldsIdTargeted", baseTableFieldsIdTargeted)


  // send API request
  const res = await queryIGUserPostsFilteredHashtag(headers, params)
  console.log("res-data", res.data)
  
  if (res.isError) {
    // Handle the error and end the function
    await handleErrorTip(res.errorMsg, "request-error")
    return
  } else {
    postTotalNum = res.data.total_length
    postNumFilteredHashtag = res.data.hashtag_length
  }

  // handle API response data to Target data structure or 
  const targetDataStructure = queryTargetDataStructure(res.data, baseTableFieldsIdTargeted)
  console.log("targetDataStructure", targetDataStructure)

  // Write the data back to the multidimensional table
  await writeDataBackToTable(table, targetDataStructure)

  // 插件运行结束提示
  await showProcessTip("end")

}


/**
 * @query(check) {检查是否已填写了必要信息}
 * @return {object}
 */
const checkIsRequiredInfoFilled = async () => {
  // 检查是否填写了 Instagram 请求头信息
  if (!IG_COOKIE.value || !IG_APP_ID.value || !IG_CLAIM.value) {
    await handleErrorTip("", "empty_headers")
    return {"isError": true}
  }


  // 检查是否填写了 Instagram 请求参数信息
  if (!targetUsername.value || !targetHashTag.value) {
    await handleErrorTip("", "empty_params")
    return {"isError": true}
  }

  // 检查是否已选择了返回字段
  if (responseFieldsSelected.value.length === 0) {
    await handleErrorTip("", "empty_response_fields_selected")
    return {"isError": true}
  }

  return {"isError": false}
}

/**
 * @query(status) {插件运行状态通知}
 * @param {string} Tiptype 提示类型
 */
const showProcessTip = async (Tiptype) => {
  if (Tiptype === "start") {
    await bitable.ui.showToast({
      toastType: 'success',
      message: t('infoTip.start')
    })
  } else if (Tiptype === "end") {
    let endInfo = t('infoTip.end_sentence')
    const resDesc = endInfo.replace("postTotalNum", postTotalNum).replace("postNumFilteredHashtag", postNumFilteredHashtag)
    processResultDesc.value = resDesc
    
    await bitable.ui.showToast({
      toastType: 'success',
      message: t('infoTip.end')
    })
  }
}

/**
 * @query {获取 SDK table、view、existedFieldMetaList 等信息}
 * @return {obj} table, view, existedFieldMetaList
 */
const queryBaseTableAndView = async () => {
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  const existedFieldMetaList = await table.getFieldMetaList();

  return {table, view, existedFieldMetaList}
}



/** @query{返回 Instagram Cookie 等请求头信息}
 * @return{object}
 */
const queryIGHeaderInput = () => {


  localStorage.setItem('IG_COOKIE', IG_COOKIE.value)   // string 类型
  localStorage.setItem('IG_APP_ID', IG_APP_ID.value)   // string 类型
  localStorage.setItem('IG_CLAIM', IG_CLAIM.value)   // string 类型

  return {
    [IG_HEADER_STRUCTURE.cookie]: IG_COOKIE.value,
    [IG_HEADER_STRUCTURE.app_id]: IG_APP_ID.value,
    [IG_HEADER_STRUCTURE.claim]: IG_CLAIM.value
  }
}


/** @query{查询 Instagram Hashtag 等请求参数信息}
 * @return{objecy}
 */
const queryIGParamsInput = () => {
  localStorage.setItem('targetHashTag', targetHashTag.value)   // string 类型
  localStorage.setItem('targetUsername', targetUsername.value)   // string 类型

  return {
    [IG_PARAMS_STRUCTURE.hashtag]: targetHashTag.value,
    [IG_PARAMS_STRUCTURE.username]: targetUsername.value
  }
}

/** @query {查询需要返回的 fields 数组}
 * @return {array}
 */
const queryTargetResponseFields = () => {
  localStorage.setItem('responseFieldsSelected', JSON.stringify(responseFieldsSelected.value))  // object 类型

  return responseFieldsSelected.value
}


/**
 * @query {查询用户所勾选字段，对应的多维表格 fieldId}
 * @param {object} table 多维表格 SDK table 对象
 * @param {object} currentFields 当前表格的字段元信息
 * @param {array} targetFields 所勾选的目标字段
 * @return {object} simplyName-fieldId
 */
const queryBaseTableFieldsIdTargeted = async (table, existedFieldMetaList, responseFieldsSelected) => {
  // 匹配已有的字段
  const existedFieldIdObjectToWritten = getExistedFieldIdObjectToWritten(existedFieldMetaList, responseFieldsSelected)
  
  if (existedFieldIdObjectToWritten.isError) {// 错误处理，提示格式错误 
    await handleErrorTip(existedFieldIdObjectToWritten.errorMsg, "field-type-error")
    return
  }

  // 创建缺少的字段
  let baseTableFieldsIdTargeted = await createFields(existedFieldIdObjectToWritten.data, table)

  return baseTableFieldsIdTargeted
}

/**
* @query {获取要写入数据的字段的 name-id 对象，若需创建，则id值设置为-1}
* @param {array} existedFieldMetaList 表格中已经存在的字段元数据列表
* @param {array} fieldNameListToBeWritten 需要写入数据的字段名称列表
*/
const getExistedFieldIdObjectToWritten = (existedFieldMetaList, fieldNameListToBeWritten) => {
  const existedFieldIdObject = {};
  for (let field of fieldNameListToBeWritten) {
    // 查找与fieldNameListToBeWritten相匹配的existedFieldMetaList项目
    const foundField = existedFieldMetaList.find(f => f.name ===  t(`${BASE_I18N_FIELD_PATH}.${field}`));

    const type = getFieldTypeByName(field)
    const checkTip = `「${t(`${BASE_I18N_FIELD_PATH}.${field}`)}」${t(`${BASE_I18N_FIELD_TYPE_CHECK_PATH}.${type}`)}`

    console.log("foundField", foundField)
    if (field.endsWith('Count') && foundField && foundField.type !== 2)
      return {errorMsg: checkTip, isError: true}
    else if ( (field == t('selectGroup.text') || field.endsWith('Link') ) && foundField && foundField.type !== 1)
      return {errorMsg: checkTip, isError: true}
    else if ( (field == t('selectGroup.createTime') || field == t('selectGroup.requestTime') ) && foundField && foundField.type !== 5)
      return {errorMsg: checkTip, isError: true}
    

    
    // 如果找到了相应的项目，就使用其id，否则设置为-1
    existedFieldIdObject[field] = foundField ? foundField.id : -1;
  }

  return {data: existedFieldIdObject, isError: false};
} 

/**
* @common {给fieldIdObjectToWritten中所有id标记为-1的field创建字段，并返回其id}
* @param {object} fieldIdObjectToWritten 需要写入数据的字段 name-id 集合
* @param {object} table 表格实例
*/
const createFields = async (fieldIdObjectToWritten, table) => {
  const oldFieldIdObjectToWritten = JSON.parse(JSON.stringify(fieldIdObjectToWritten))
  const newFieldIdObjectToWritten = {...oldFieldIdObjectToWritten}

  for (let key in oldFieldIdObjectToWritten) {
    if (oldFieldIdObjectToWritten[key] == -1) {
      const name = t(`${BASE_I18N_FIELD_PATH}.${key}`)
      const type = getFieldTypeByName(key)

      newFieldIdObjectToWritten[key] = await table.addField({
        type: FieldType[type],
        name,
      })
    }
  }

  return newFieldIdObjectToWritten
}

/**
* @query {依据name返回field对应的字段类型}
* @param {string} name fieldName
* @return {string} fieldType
*/
const getFieldTypeByName = (name) => {
  if (TEXT_FIELD_ARRAY.includes(name))
    return "Text"
  else if (DATETIME_FIELD_ARRAY.includes(name)) 
    return "DateTime"
  else if (NUMBER_FIELD_ARRAY.includes(name))
    return "Number"

  return "Text"
}



/** @query{查询 Instagram 用户特定 Hastag 下的全部帖子}
 * @param {object} headers IG 请求头信息
 * @param {object} params IG 请求参数信息
 * @return {object} 通过 isError 字段标记请求是否成功
 */
const queryIGUserPostsFilteredHashtag = async (headers, params) => {
  const headersCopy = JSON.parse(JSON.stringify(headers))
  const paramsCopy = JSON.parse(JSON.stringify(params))
  headersCopy[IG_HEADER_STRUCTURE.cookie] = headersCopy[IG_HEADER_STRUCTURE.cookie].replaceAll('"', '\\"')
  
  const requestData = {...headersCopy, ...paramsCopy}

  // 发送POST请求
  try {
      const response = await axios.post(`${BASE_REQUEST_URL}/get_user_total_posts`, requestData);
      console.log('Response:', response.data);
      return { data: response.data, isError: false };
  } catch (error) {
      console.error('Error:', error);
      return { errorMsg: error.response.data.error, isError: true };
  }
}

/**
 * @query(hanlde) {报错提示函数}
 * @param {string} errorMsg 错误提示
 * @param {string} errorType 错误类型
 */
const handleErrorTip = async (errorMsg, errorType) => {
  if (errorType === "request-error" || errorType === "field-type-error") {
    await bitable.ui.showToast({
      toastType: 'error',
      message: errorMsg
    })
  }

  else {
    await bitable.ui.showToast({
      toastType: 'error',
      message: t(`errorTip.${errorType}`)
    })
  }
}

/**
 * @query {获取格式化的 fieldId-value 数据}
 * @param {object} response 接口返回的 IG 数据
 * @param {object} baseTableFieldsIdTargeted simplyName-fieldId
 * @return {array} 格式化的 fieldId-value 数据
 */
const queryTargetDataStructure = (response, baseTableFieldsIdTargeted) => {
  const data = JSON.parse(JSON.stringify(response.res))
  const fieldIdObj = JSON.parse(JSON.stringify(baseTableFieldsIdTargeted))
  const mapRelationship = {
    "postLink": "code",
    "videoViewCount": "video_view_count",
    "likeCount": "like_count",
    "commentCount": "comment_count",
    "createTime": "created_at",
    "text": "text",
    "videoLink": "video",
  }
  let list = []

  for (let i = 0; i < data.length; i++) {
    let obj = {}
    for (let key in fieldIdObj) {
      if (key === "requestTime") {
        const now = new Date();
        obj[fieldIdObj[key]] = now.getTime();
        continue
      }

      let value = data[i][mapRelationship[key]]

      if (key === "postLink") 
        value = "https://www.instagram.com/p/" + value
      else if (key === "createTime")
        value = value*1000


      obj[fieldIdObj[key]] = value
    }
    
    list.push({fields: obj})
  }

  return list
}

/**
 * 将 IG 数据写回多维表格
 * @param {object} table SDK table 实例
 * @param {array} targetDataStructure 特定数据结构的列表，每个元素为  fieldId-value object
 */
const writeDataBackToTable = async(table, targetDataStructure) => {
  const res = await table.addRecords(targetDataStructure);
}

/**
 * @common(set) {读取缓存，并赋值给相关变量}
 */
const setVariableFromLocalStorage = () => {
  if (localStorage.getItem('responseFieldsSelected') !== null) {  // object 类型
    responseFieldsSelected.value = JSON.parse(localStorage.getItem('responseFieldsSelected')) 
  }

  if (localStorage.getItem('IG_COOKIE') !== null) {  // string 类型
    IG_COOKIE.value = localStorage.getItem('IG_COOKIE')
  }
  if (localStorage.getItem('IG_APP_ID') !== null) {  // string 类型
    IG_APP_ID.value = localStorage.getItem('IG_APP_ID')
  }
  if (localStorage.getItem('IG_CLAIM') !== null) {  // string 类型
    IG_CLAIM.value = localStorage.getItem('IG_CLAIM')
  }
  

  if (localStorage.getItem('targetHashTag') !== null) {  // string 类型
    targetHashTag.value = localStorage.getItem('targetHashTag')
  }
  if (localStorage.getItem('targetUsername') !== null) {  // string 类型
    targetUsername.value = localStorage.getItem('targetUsername')
  }
}


/**
 * @common(handle) {处理 checkbox 全选点击事件}
 * @param {boolean} val checkbox 是否全选
 */
const handleselectAllFieldsChange = (val) => {
  const data = JSON.parse(JSON.stringify(responseFieldsAvaiable.value))
  if (val) {
    for (const item of data)
      responseFieldsSelected.value.push(item.label);


  } else {
    responseFieldsSelected.value = []
  }
  isIndeterminateToMap.value = false
}

/**
 * @common(handle) {处理 checkbox 点击事件}
 * @param {array} value 
 */
const handleresponseFieldsSelectedChange = (value) => {
  const checkedCount = value.length
  selectAllFields.value = checkedCount === responseFieldsAvaiable.value.length
  isIndeterminateToMap.value = checkedCount > 0 && checkedCount < responseFieldsAvaiable.value.length
  console.log('responseFieldsSelected:', responseFieldsSelected.value)

}
  

onMounted(async () => {
  // 初始化勾选字段
  console.log("onMounted >> 已选中的返回字段数组", responseFieldsSelected.value)

  // 读取缓存数据，并赋值给变量
  setVariableFromLocalStorage()
  

});
    
</script>



<style scoped>
:deep(.el-icon) {
  color: #3370ff !important;
}   
</style>
