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
          v-model="selectedTransactionId"
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
      <el-dialog
        v-model="availableForSaleVisible"
        title="最优卖出份额及其最大收益为："
        width="400"
      >
      <div>建议卖{{result.optimalShares}}份</div>
      <div>此时的收益为最大：{{result.maxProfit}}</div>
        <template #footer>
          <div class="dialog-footer">
            <el-button @click="availableForSale = false">取消</el-button>
          </div>
        </template>
      </el-dialog>
      <template #footer>
        <el-button @click="buyIt">买入</el-button>
        <el-button @click="exportToExcel(newTransaction.name)">导出表格</el-button>
        <el-button @click="calculatingSale">计算卖出收益</el-button>
      </template>
    </el-card>

    <el-table :data="theTransac" border style="width: 100%">
      <el-table-column prop="value" label="净值" width="78" />
      <el-table-column prop="share" label="份额" width="80" />
      <el-table-column label="交易日期" width="300">
        <template #default="scope">
          <span>{{ dataFormat(scope.row.time) }}</span>
        </template>
      </el-table-column>
    </el-table>
 
    <div>
  </div>
  </div>
</template>

<script>

import { defineComponent, onMounted, watch, ref } from 'vue'
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
    const selectedTransactionId = ref(null)
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
    const availableForSaleVisible = ref(false)
    const headers = ref([])
    const result = ref({
      optimalShares: 0,
      maxProfit: 0
    })
    const theTransac = ref([])

    // 监听 selectedTransactionId 的变化
    watch(() => selectedTransactionId.value, (newVal, oldVal) => {
      if (newVal !== oldVal) {
        for(let i of transactionData.value){
          if(i[0].id === selectedTransactionId.value){
            theTransac.value = i
          }
        }
      }
    }, { immediate: true })
    // 同步 selectedTransactionId 和 newTransaction.id
    watch(() => selectedTransactionId.value, (newVal) => {
      newTransaction.value.id = newVal
    }, { immediate: true }); // 立即执行一次以同步初始值

    const dataFormat = (time)=> {
      time = new Date(parseFloat(time))
      const year = time.getFullYear()
      const month = time.getMonth()
      const day = time.getDate()
      return `${year}-${month + 1}-${day}`
    }

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

    // 增加基金
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

    // 增加买入记录
    const buyIt = () => {
      // 先拉取该基金的表格
      // 将基金加入transactionData数组
      // 筛选transactionData中对应的name，再push
      let findId = false
      for(let i of transactionData.value){
        if(i[0] && i[0].id === newTransaction.value.id){
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

    // 导出数据表
    const exportToExcel = () => {
      // 将数据转换为工作表
      // 将buy和sell字段从数组转换为字符串
      fundData.value = fundData.value.map(fund => ({
        ...fund,
        buy: fund.buy,
        sell: typeof fund.sell === 'string' ? fund.sell : fund.sell.join(',')
      }))
      let ws = XLSX.utils.json_to_sheet(fundData.value)
      // 创建一个新的工作簿并将工作表添加到其中
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'Sheet1')
      console.log('transactionData', transactionData.value)
      // 遍历transactionData的数组
      if (transactionData.value[0][0]){
        for(let i of transactionData.value){
          ws = XLSX.utils.json_to_sheet(i)
          XLSX.utils.book_append_sheet(wb, ws, i[0].id)
        }
      } 
      // 生成Excel文件并触发下载
      XLSX.writeFile(wb, 'holdingFund.xlsx')
    }

    // 解析手续费规则
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
      return fees
    }
    
    // 计算最优卖出份额及其最大收益
    const calculateOptimalSell = (transactions, currentPrice, currentDate, feeRules) => {
      // 将日期字符串转换为日期对象
      const parseDate = (dateStr) => new Date(parseFloat(dateStr)).setHours(0, 0, 0, 0)
      
      // 计算持有天数
      const calculateHoldingDays = (buyDate, sellDate) => {
          const timeDiff = sellDate - buyDate
          return Math.floor(timeDiff / (1000 * 60 * 60 * 24)) // 转换为天数
      }

      // 按 FIFO 原则计算当前持有份额
      const currentHoldings = []
      for (const transaction of transactions) { // 遍历交易记录
        const { time, share, value } = transaction
        const transactionDate = parseDate(time)
        if (share > 0) {  // 买入份额
          currentHoldings.push({ time: transactionDate, share, value })
        } else { // 卖出份额（按 FIFO 扣除）
          let remainingShares = -share // 剩余扣除份额
          while (remainingShares > 0 && currentHoldings.length > 0){
            const earliestHolding = currentHoldings[0] // 先取最早的份额
            if (earliestHolding.share > remainingShares) {
              earliestHolding.share -= remainingShares
              remainingShares = 0
            } else {
              remainingShares -= earliestHolding.share
              currentHoldings.shift() // 移除已完全卖出的份额
            }
          }
        }
      }
      
      // 计算平均买入成本
      const totalBuyShares = currentHoldings.reduce((sum, h) => sum + parseFloat(h.share), 0) // 目前所持有的全部份额
      const totalBuyCost = currentHoldings.reduce((sum, h) => sum + h.share * h.value, 0) // 买所有份额的花费
      const avgCost = totalBuyCost / totalBuyShares

      // 根据持有天数和手续费规则分组
      const currentDateObj = currentDate.setHours(0, 0, 0, 0)
      const groupedShares = feeRules.map(rule => ({
        ...rule,
        shares: 0, // 初始化份额
      }))
      // 分组计算收益
      for (const holding of currentHoldings) {
        const holdingDays = calculateHoldingDays(holding.time, currentDateObj)
        for (const group of groupedShares) { // 根据阶梯分组计算
          const {left, right} = group
          if (holdingDays >= left && (!right || holdingDays < right)) {
            group.shares += parseFloat(holding.share)
            break; // 找到匹配的组后跳出循环
          }
        }
      }
      // 计算不同卖出份额的净收益
      let maxProfit = -Infinity
      let optimalShares = 0

      // 卖出全部份额
      const totalShares = groupedShares.reduce((sum, group) => sum + group.shares, 0)
      for (let q = 0; q <= totalShares; q++) {
        let remainingShares = q
        let profit = 0
        // 按手续费从低到高的顺序卖出
        for (const group of groupedShares.sort((a, b) => a.rate - b.rate)) {
          const sellShares = Math.min(remainingShares, group.shares) 
          profit += sellShares * currentPrice * (1 - group.rate) // 扣除手续费
          remainingShares -= sellShares
          if (remainingShares === 0) break // 卖出完毕
        }

        // 扣除成本
        profit -= q * avgCost
        // 更新最优解
        if (profit > maxProfit) {
          maxProfit = profit
          optimalShares = q
        }
      }
      return {
        optimalShares,
        maxProfit: parseFloat(maxProfit.toFixed(2)) // 保留两位小数
      }
    }

    const calculatingSale = () => {
      availableForSaleVisible.value = true
      let theTransactionData
      // 从transactionData找到匹配newTransaction.id的买入记录列表
      for(let i of transactionData.value){
        if(i[0].id === newTransaction.value.id){
          theTransactionData = i
        }
      }
      // 获取卖出手续费计算规则
      const sellFeeRateRuleJson = fundData.value.find(item => item.id === newTransaction.value.id).sell
      const sellFeeRateRule = parseFeeRules(sellFeeRateRuleJson)
      result.value = calculateOptimalSell(theTransactionData, parseFloat(newTransaction.value.value), new Date(), sellFeeRateRule)
    }
    onMounted(async () => {
    })
    return {
      store,
      buyIt,
      theTransac,
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
      result,
      availableForSaleVisible,
      selectedTransactionId,
      dataFormat,
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
