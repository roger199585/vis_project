<template>
    <v-layout row nowrap fill-height style="overflow:hidden">
        <v-flex style="padding:0px;padding-right:10px;width:50px">
            <!-- 明确 canvas 的长宽,防止出现比例不一致造成的扭曲问题 -->
            <!-- 千萬注意，style 設定的px 跟 canvas 所在座標系並不是一一對應的 -->
            <canvas ref="color_line" width=45 height=300>
                can not show canvas color line
            </canvas>
            <!-- <v-btn v-if='!loading' fab dark small color="indigo">
                <v-icon @click='autoColResize()' dark>add</v-icon>
            </v-btn> -->
            <v-progress-circular
            indeterminate
            color="primary"
            v-if='loading'
            ></v-progress-circular>
        </v-flex>
        <v-flex lg12 style="padding:0px;overflow:hidden">        
            <hot-table ref="hot" :settings="settings"></hot-table>
        </v-flex>
    </v-layout>
</template>
<script>
import { HotTable } from '@handsontable/vue';
const hotInstance = {}
let vm = undefined
let d3 = undefined
export default {
    components: {
        HotTable
    },
    data: () => {
        return {
            loading:true,
            settings: {
                data: [
                    //資料格式
                    // {
                    //     'date':'2018/03/05',
                    //     'Ford':"abad",
                    // },
                ],
                columns:[
                    //欄位設定
                    // {
                    //     data:'date',
                    //     type:'numeric',
                    // },
                ],
                colHeaders: [
                    //欄位名稱設定
                    // 'date',
                    // 'Ford',
                ],
                rowHeaders: [],
                columnSorting:false,
                // 開啓了虛擬渲染行，handsontable在渲染的時候爲了效能，只會渲染出窗口範圍內的內容。
                // 開啓後，會保留之前渲染過的內容，但是依舊無法在一開始就直接加載全部的內容
                renderAllRows: false,
                // 开启只读模式,不允许修改内容
                readOnly: true,
                // columnsSize 設定的是所有cell的寬度
                autoColumnSize: false,
                rowHeaderWidth: 125,
                // rowSize 設定的是所有cell的高度
                autoRowSize:false,
                currentRowClassName: 'currentRow',
                stretchH: 'none',
                // 右鍵菜單設定
                contextMenu:false,
                // 只允许使用单独选定一个 cell
                selectionMode:'single',
                // afterRenderer 在每一個data cell 繪製完成後都會觸發一次
                afterRenderer:(TD,row,column,prop,value) => {
                    //////////////// 
                    // api 說明, TD 指向這個 cell, value 指的是這個這個 cell 的數值,
                    // row 表示當前行數, column 表示當前的列數
                    ////////////////
                    // 修改文字颜色
                    // TD.style.color = 'blue'
                    // 修改cell背景色
                    TD.style.backgroundColor = vm.color(value)

                },
                afterOnCellMouseDown:(event,coords) => {
                    let row = coords.row
                    let col = coords.col
                    // if(row !== col){
                    //     let rowLabel = vm.columns[row]
                    //     let colLabel = vm.columns[col]
                    //     vm.EventBus.root.loadScatterplot(rowLabel,colLabel)
                    // }
                    let rowLabel = vm.columns[row]
                    let colLabel = vm.columns[col]
                    vm.EventBus.root.loadScatterplot(rowLabel,colLabel)
                },
            },
            color_range:['moccasin', 'rgba(147,210,197,1)']
        };
    },
    created(){
        let vm = this
        vm.extent = undefined
        vm.color = undefined
    },
    methods:{
        // 給定資料欄位名稱
        setHeaders(columns){
            vm.settings.colHeaders = columns
            vm.settings.rowHeaders = columns
            vm.columns = columns
            vm.colSetting(columns)
        },
        // 改變資料
        changeData(data,extent){
            vm.settings.data = data
            vm.extent = extent
            vm.color = d3.scaleLinear()
                        .range(vm.color_range)
                        .domain([extent[0],extent[1]])

        },
        // 清除資料
        clearData(){
            vm.settings.data = []
            vm.settings.columns = []
            vm.settings.colHeaders = []
            vm.settings.rowHeaders = []
        },
        // 将表格中所有类型都设定为浮点数
        colSetting(columns){
            columns.forEach(() => {
                let temp = {}
                temp.type = 'numeric'
                temp.numericFormat = {
                    pattern: '0.000'
                }
                vm.settings.columns.push(temp)
            })
        },
        // 繪製色碼條
        drawColorScale(){
            let canvas = vm.$refs.color_line
            if (canvas.getContext) {
                let ctx = canvas.getContext('2d');
                // 設定 gradient 的時候設定是 x1，y1，x2，y2
                let gradient = ctx.createLinearGradient(0,20,0,280)
                gradient.addColorStop(1,'moccasin')
                gradient.addColorStop(0,'rgba(147,210,197,1)')

                ctx.font = "12px serif"
                ctx.fillText("Min",17,295)
                ctx.fillText("Max",17,15)
                
                ctx.fillStyle = gradient
                // 注意繪製矩形的時候輸入是 x，y，width，height
                ctx.fillRect(12,20, 30,260)
            }
        },
        // 自動擴展欄位
        autoColResize(){
            let hot = vm.$refs.hot.hotInstance
            vm.settings.autoColumnSize = !vm.settings.autoColumnSize
            hot.render()
        }

    },
    mounted(){  
        vm = this
        d3 = vm.$d3
        hotInstance.root = vm
        vm.load = true 
        vm.drawColorScale()
    }
}
</script>


<style>
/* .currentRow {
    background-color: #F9F9FB !important;
} */
.relative{
    background: white;
}
tr th:first-child{
    background: white;
    padding: 0;
}
.rowHeader {
    width: 180px;
}
::-webkit-scrollbar-track {
  background: #fafafa; 
}

.v-progress-circular{
    margin: 0.6rem
}

</style>

<style src="./css/handsontable.full.css"></style>