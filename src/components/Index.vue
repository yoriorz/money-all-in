<template>
  <div class="page">
    <header-top :option="heardOption"></header-top>
    选择基金：
    <input type="file" @change="handleFileUpload" accept=".xlsx, .xls" />
    <div v-if="excelData.length">
      <h3>Excel Data:</h3>
      <div>{{ excelData.length }}</div>
    </div>
    <div>请填写今日基金净值</div>
    <div>基金名：<input type="text"></div>
    <div>今日基金净值：<input type="text"></div>
    <div>日期：</div>
    <div>
      <Button type="primary" @click="sellIt">卖出</Button>
      <Button type="primary" @click="buyIt">买出</Button>
    </div>
    <div>
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Age</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="person in people" :key="person.id">
          <td>{{ person.name }}</td>
          <td>{{ person.age }}</td>
        </tr>
      </tbody>
    </table>
    <button @click="exportToExcel">Export to Excel</button>
  </div>
  </div>
</template>

<script>

import { defineComponent, onMounted, ref } from 'vue'
import { useStore } from 'vuex'
import { useRouter } from 'vue-router'
import * as XLSX from 'xlsx'

export default defineComponent({
  name: 'Index',
  data: function () {
    return {
      // 导航栏信息
      heardOption: {
        centerTitle: '这里是导航栏'
      }
    }
  },
  setup () {
    const store = useStore()
    const router = useRouter()
    const excelData = ref([]) // 用于存储解析后的Excel数据
    const headers = ref([])
    const handleFileUpload = (event) => {
      const file = event.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result)
        const workbook = XLSX.read(data, { type: 'array' })
        // 假设读取第一个工作表
        const firstSheetName = workbook.SheetNames[0]
        const worksheet = workbook.Sheets[firstSheetName]
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 }) // 将工作表转换为JSON
        // 更新表格数据和表头
        headers.value = jsonData[0] ? Object.keys(jsonData[0]) : []
        excelData.value = jsonData.slice(1) // 跳过表头行
        // 存储解析后的数据
        console.log('handleFileUpload', headers.value, excelData.value)
      }
      reader.readAsArrayBuffer(file)
    }

    const buyIt = () => {
      // 计算是否适合买入
      // 从文件夹里拉数据
      // 然后计算当前盈亏
    }

    const sellIt = () => {
      // 计算是否适合卖出
      // 从文件夹里拉数据
      // 然后计算当前盈亏
    }
    const people = ref([
      { id: 1, name: 'John Doe', age: 30 },
      { id: 2, name: 'Jane Smith', age: 25 }
      // ...更多数据
    ])
    const exportToExcel = () => {
      // 将数据转换为工作表
      const ws = XLSX.utils.json_to_sheet(people.value)
      // 创建一个新的工作簿并将工作表添加到其中
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'Sheet1')
      // 生成Excel文件并触发下载
      XLSX.writeFile(wb, 'people.xlsx')
    }
    return {
      store,
      buyIt,
      excelData,
      people,
      exportToExcel,
      handleFileUpload,
      sellIt
    }
  }
})
</script>

<style lang="less" scoped>
.page {
  width: 100%;
  height: 100%;
  .result {
    margin: 200px auto;
    width: 400px;
    height: 80px;
    line-height: 80px;
    background-color: yellowgreen;
    border-radius: 20px;
    text-align: center;
  }
}
</style>
