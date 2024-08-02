#### 官网：

https://echarts.apache.org/handbook/zh/get-started/

#### 下载

`npm install echarts --save`

#### 在main.ts中引入全局挂载

```ts
//echarts，也可只在单个vue文件中import而不挂载
import * as echarts from 'echarts'
app.config.globalProperties.$echarts = echarts
```

使用：

1. 定义有固定高度宽度的div来显示图形
2. 在div上绑定ref属性
3. 引用，初始化ecahrts
4. 配置数据
5. 挂载使用函数图形

```vue
<template>
  <el-card style="margin-left: 20px;flex: 1;">
    <template #header>
      <div class="card-header">
        <span>热门商品</span>
      </div>
    </template>
    <div ref="myChartRef" :style="{width:'400px',height:'300px'}"></div>
  </el-card>
</template>

<script setup lang="ts">
// import * as echarts from 'echarts';
import {nextTick, onMounted, reactive, ref} from "vue";
import useInstance from '@/hooks/useInstance'

const {global} = useInstance()
const mainHeight = ref(0)
const myChartRef = ref<HTMLDivElement>()

//图形
const chart = () => {
  //初始化，传递ref
  const chartInstance = global.$echarts.init(myChartRef.value)
  //配置项
  let option = reactive({
    title: {
      subtext: 'Fake Data',
      left: 'center'
    },
    tooltip: {
      trigger: 'item'
    },
    xAxis: {
      data: ["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"]
    },
    yAxis: {
      type: 'value'
    },
    series: [{
      name: '销量',
      type: 'bar',
      data: [5, 20, 36, 10, 10, 20]
    }]
  });
  //动态获取数据通过axios发送请求获取数据，设置到option的data中
	
  chartInstance.setOption(option)
}

onMounted(()=>{
  chart();
  nextTick(()=>{
    mainHeight.value=window.innerHeight-100
  })
})
</script>

<style scoped lang="scss"></style>
```



