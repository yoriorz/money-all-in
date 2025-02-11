<template>
  <div class="page">
    <el-card style="max-width: 480px; margin-bottom: 20px">
      <template #header>
        <div class="card-header">
          <span>请导入基金数据：</span><input type="file" @change="handleFundDataFileUpload" accept=".xlsx, .xls" />
        </div>
      </template>
      <el-table :data="fundData" border style="width: 100%">
        <el-table-column prop="id" label="代号" width="78" />
        <el-table-column prop="name" label="基金名" width="300" />
      </el-table>
      <el-dialog
        v-model="dialogVisible"
        title="请输入新增基金"
        width="400"
      >
        <el-input v-model="newFund.id" style="width: 240px" placeholder="请输入代号" />
        <el-input v-model="newFund.name" style="width: 240px" placeholder="请输入基金名" />
        <el-input v-model="newFund.buy" style="width: 240px" placeholder="请输入买入手续费" />
        <el-input v-model="newFund.sell1" style="width: 240px" placeholder="请输入第一区间卖出手续费" />
        <el-input v-model="newFund.sell2" style="width: 240px" placeholder="请输入第二区间卖出手续费" />
        <el-input v-model="newFund.sell3" style="width: 240px" placeholder="请输入第三区间卖出手续费" />
        <template #footer>
          <div class="dialog-footer">
            <el-button @click="dialogVisible = false">取消</el-button>
            <el-button type="primary" @click="addFund">
              确定
            </el-button>
          </div>
        </template>
      </el-dialog>
      <template #footer>
        <el-button plain @click="dialogVisible = true">新增</el-button>
        <el-button @change="deleteFund">删除</el-button>
        <el-button @click="exportToExcel('Sheet1')">导出表格</el-button>
      </template>
    </el-card>

    <el-card style="max-width: 480px; ; margin-bottom: 20px">
      <template #header>
        <div class="card-header">
          <span>请填写今日基金数据：</span>
        </div>
      </template>
      <div>基金名：
        <el-select
          v-model="newTransaction.id"
          placeholder="请选择基金文件"
          style="width: 240px"
        >
          <el-option
            v-for="item in fundData"
            :key="item.id"
            :label="item.name"
            :value="item.id"
          />
        </el-select>
      </div>
      <div>基金净值：<el-input v-model="newTransaction.value" style="width: 229px" placeholder="请输入净值" /></div>
      <div>买入/卖出份额：<el-input v-model="newTransaction.share" style="width: 195px" placeholder="请输入份额" /></div>
      <div class="block">
          <div class="demonstration">交易日期:
            <el-date-picker
              v-model="newTransaction.time"
              type="datetime"
              placeholder="选择日期"
              format="YYYY/MM/DD"
              value-format="x"
            />
          </div>
        </div>
      <div>
        <el-button @click="sellIt">卖出</el-button>
        <el-button @click="buyIt">买入</el-button>
        <br>
        <el-button @click="exportToExcel(newTransaction.name)">导出表格</el-button>
        <el-button @click="calculatingSale">计算卖出收益</el-button>
      </div>
    </el-card>
 
    <div>
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
    }
  },
  components: {
  },
  setup () {
    const store = useStore()
    const fundData = ref([])
    const transactionData = ref([])
    const newTransaction = ref({
      id: '',
      value: '',
      share: '',
      time: ''
    })
    const excelData = ref([]) // 用于存储解析后的Excel数据
    const newFund = ref({
      id:'',
      name:'',
      buy:[],
      sell:[],
      sell1: '',
      sell2: '',
      sell3: ''
    })
    const dialogVisible = ref(false)
    const headers = ref([])

    // 获取基金数据
    const handleFundDataFileUpload = async (event) => {
      const file = event.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result)
        const workbook = XLSX.read(data, { type: 'array' })
        // 假设第一个工作表,获取现有基金的列表
        let firstSheetName = workbook.SheetNames[0]
        let worksheet = workbook.Sheets[firstSheetName]
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 }) // 将工作表转换为JSON
        // 更新表格数据和表头
        headers.value = jsonData[0] ? Object.keys(jsonData[0]) : []
        excelData.value = jsonData.slice(1) // 跳过表头行
        fundData.value = excelData.value.map(item => ({
          id: item[0],
          name: item[1],
          buy: item[2],
          sell: item[3].match(/\(([^)]+)\),\s*([\d.]+)%/g)
        }))

        // 读取后面的sheetnames，获取每个基金的交易数据，每个交易数据都要读取，以免被新数据覆盖了
        if(workbook.SheetNames.length > 1){
          let SheetName
          // 遍历所有的SheetName，获取交易数据
          for(let i = 1; i < workbook.SheetNames.length; i++){
            SheetName = workbook.SheetNames[i]
            worksheet = workbook.Sheets[SheetName]
            let transactionJsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 }) // 将工作表转换为JSON
            transactionJsonData.value = transactionJsonData.slice(1) // 跳过表头行
            transactionJsonData.value = transactionJsonData.value.map(item => ({
              id: item[0],
              value: item[1],
              share: item[2],
              time: item[3]
            }))
            transactionData.value.push(transactionJsonData.value)
          }
        }
      }
      reader.readAsArrayBuffer(file)
    }

    const addFund = () => {
      dialogVisible.value = false
      newFund.value.sell = [newFund.value.sell1, newFund.value.sell2, newFund.value.sell3].filter(value => value != null && value !== "")
      delete newFund.value.sell1
      delete newFund.value.sell2
      delete newFund.value.sell3
      // 给fundData里增加newFund基金
      fundData.value.push(newFund.value)
      newFund.value = {
        id:'',
        name:'',
        buy:[],
        sell:[],
        sell1: '',
        sell2: '',
        sell3: ''
      }
    }

    const deleteFund = () => {
      // 给fundData里删除一个基金
    }

    const buyIt = () => {
      // 先拉取该基金的表格
      // 将基金加入transactionData数组
      // 筛选transactionData中对应的name，再push
      let findId = false
      for(let i of transactionData.value){
        if(i[0].id === newTransaction.value.id){
          findId = true
          i.push(newTransaction.value)
        }
      }
      if(!findId){
        transactionData.value.push([{
          id: newTransaction.value.id,
          value: newTransaction.value.value,
          share: newTransaction.value.share,
          time: newTransaction.value.time
        }])
      }
      newTransaction.value = {
        id: newTransaction.value.id,
        value: '',
        share: '',
        time: ''
      }
    }

    const sellIt = () => {
      // 计算是否适合卖出
      // 从文件夹里拉数据
      // 然后计算当前盈亏
    }
    const holdingFund = ref([
      { id: '007467', name: '华泰柏瑞中证红利低波动ETF联接C',
      buy:['0'],
      sell:['(t>=0,t<7),1.5%','(t>=7,t<30),0.1%','(t>=30),0%']
      },
      { id: '000307', name: '易方达黄金ETF联接A',
        buy:['0.7%'],
        sell:['(t>=0,t<7),1.5%','(t>=7,t<365),0.2%','(t>=365,t<730),0.05%','(t>=730),0%']
      },
      { id: '011854', name: '招商中证消费龙头指数增强C',
        buy:['0'],
        sell:['(t>=0,t<7),1.5%','(t>=7,t<30),0.1%','(t>=30),0%']
      },
      { id: '001632', name: '天弘中证食品饮料ETF链接C',
        buy:['0'],
        sell:['(t>=0,t<7),1.5%','(t>=7),0%']
      },
      { id: '022463', name: '富国中证A500ETF联接A',
        buy:['1.2%'],
        sell:['(t>=0,t<7),1.5%','(t>=7),0%'] },
      { id: '161027', name: '富国中证全指证券公司指数（LOF）A',
        buy:['1.2%'],
        sell:['(t>=0,t<7),1.5%','(t>=7),0.5%'] }
    ])

    const exportToExcel = () => {
      // 将数据转换为工作表
      // 将buy和sell字段从数组转换为字符串
      fundData.value = fundData.value.map(fund => ({
        ...fund,
        buy: fund.buy,
        sell: fund.sell.join(',')
      }))
      let ws = XLSX.utils.json_to_sheet(fundData.value)
      // 创建一个新的工作簿并将工作表添加到其中
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'Sheet1')
      // 遍历transactionData的数组
      for(let i of transactionData.value){
        ws = XLSX.utils.json_to_sheet(i)
        XLSX.utils.book_append_sheet(wb, ws, i[0].id)
      }
      // 生成Excel文件并触发下载
      XLSX.writeFile(wb, 'holdingFund.xlsx')
    }

    const parseFeeRules = (rules) => {
      // 创建一个数组来存储解析后的规则
      const fees = []
      for(let rule of rules) {
        // 分割出条件和费率
        const [condition, percentage] = rule.split('),')
        // 解析条件字符串，得到天数区间
        const [left, right] = condition.split(',').map(cond => {
          // 如果条件包含=，证明是left
          if(cond.includes('=')){
            const [op, value] = cond.trim().split('=') // 拆分获得 (t> 和 0
            return parseFloat(value)
          }else { // 否则则为right
            const [t, value] = cond.trim().split('<') // 则拆分为 t 和 7 或者 null
            return value - 1 ? parseFloat(value)-1:null
          }
        })      
        // 解析手续费率，去除百分号并转换为浮点数
        const rate = parseFloat(percentage.replace('%', '')) / 100
        // 将解析后的规则添加到数组中
        fees.push({ left, right, rate })
      }
      // 定义一个函数，根据天数t返回手续费率
      function getFeeRate(t) {
        // 遍历规则数组，找到第一个满足条件的区间
        for (let fee of fees) {
          if ((t >= fee.left) && (fee.right === null || t < fee.right + 1)) {
            return fee.rate
          }
        }
        
        // 如果没有找到匹配的区间，则返回一个默认值（这里假设为0%，但根据实际情况可能需要调整）
        return 0;
      }
      return getFeeRate
    }

    const calculatingSale = () => {
      let theTransactionData
      // 从transactionData找到匹配newTransaction.id的买入记录列表
      for(let i of transactionData.value){
        if(i[0].id === newTransaction.value.id){
          theTransactionData = i
        }
      }
      // 筛选出比今日净值低的数据newTransaction
      for(let i = 0; i < theTransactionData.length; i++){
        // 获取买入手续费率
        const buyFeeRate = parseFloat(fundData.value.find(item => item.id === newTransaction.value.id).buy)
        // 获取卖出手续费计算规则
        const sellFeeRateRuleJson = fundData.value.find(item => item.id === newTransaction.value.id).sell
        // 当净值低于今日净值时
        if(parseFloat(newTransaction.value.value) > parseFloat(theTransactionData[i].value)){
          // 计算今日距离买入日的时间
          const t = (new Date().setHours(0, 0, 0, 0) - new Date(theTransactionData[i].time).setHours(0, 0, 0, 0))/(1000 * 60 * 60 * 24)
          const sellFeeRate = parseFeeRules(sellFeeRateRuleJson)(t)
          console.log('sellFeeRate', sellFeeRate, new Date(theTransactionData[i].time), theTransactionData[i].time)
        }
      }

      // 计算����
      // 假设transactionData是[{id:'007467', value:['2022-01-01', '100', '1000', '2022-01-05'], share:'100', time:'0'}]
      // 假设holdingFund是[{id:'007467', buy:'0', sell:'(t>=0,t<7),1.5%'}]
      // 假设fundData是[{id:'007467', name:'华������中证红利低波动ETF联接C', buy:['0'], sell:['(t>=0,t<7),1.5%']}]
    }
    
    onMounted(async () => {
    })
    return {
      store,
      buyIt,
      excelData,
      fundData,
      exportToExcel,
      newFund,
      newTransaction,
      handleFundDataFileUpload,
      addFund,
      deleteFund,
      dialogVisible,
      calculatingSale,
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
