<template>
    <div class="pcp-wrapper" style="width:100%;height:315px;overflow-x:scroll;" ref="home"></div>
</template>

<script>
let vm = undefined
let d3 = undefined
let PIXI = undefined
export default {
	components: {},
	data() {
		return {
			
		}
    },
    created(){

    },
	methods: {  
        // pixi 初始化设定
        pixiInit(){
            vm.line_alpha = 0.3
            vm.min_axis_gap = 120
            vm.filterbox_width = 10
            vm.plot_height = 190
            vm.font = 'Arial'
            vm.font_size = 14
            vm.tick_length = 5
            vm.indicator_radius = 5
            vm.dim_slot = 120
            // 初始长宽设定
            let width = vm.$refs.home.clientWidth
            let height = vm.$refs.home.clientHeight 
            vm.app = new PIXI.Application({
                autoResize:true,
                backgroundColor:0xffffff,
                antialias:true,
                transparent: false,
                resolution:1,         
            })
            vm.app.layout = {
                'width': width,
                'height': height,
                'margin':{
                    'left':60,
                    'right':40
                }
            }
            PIXI.settings.PRECISION_FRAGMENT= 'highp'
            vm.app.renderer.roundPixels = true
            vm.app.renderer.view.style.display = 'block'            
            vm.app.renderer.resize(width,height)
            vm.$refs.home.appendChild(vm.app.view)
            // 外包装设定
            vm.wrapper = new PIXI.Container()
            vm.wrapper.name = 'PCP'
            vm.wrapper.x = 0
            vm.wrapper.y = 0
            vm.app.stage.addChild(vm.wrapper)
            // 资料折线设定
            vm.ctn_lines = new PIXI.Container()
            vm.ctn_lines.name = 'ctn_lines'
            vm.wrapper.addChild(vm.ctn_lines)
            // 轴线设定
            vm.ctn_axis = new PIXI.Container()
            vm.ctn_axis.name = 'ctn_axis'
            vm.wrapper.addChild(vm.ctn_axis)
            // 记录 filter box
            vm.filter_box = []
        },
        // 维度设定
        setDimensions(columns,data,extent){ 
            vm.columns = columns
            let length = vm.columns.length
            let margin = vm.app.layout.margin
            let latent_scatter = vm.eventBus.latent_scatter
            // 去除资料中原始包含的日期和latent信息
            vm.data = data.map(d => {
            //    return d.slice(1,-2)
                return latent_scatter.filterSelectedDimData(d)
            })
            vm.extent = extent
            vm.d3ScaleSetting(extent)
            vm.drawAxis()
            // 重新計算圖表長寬，軸線間距最小爲120px
            let slot = (vm.app.layout.width - margin.left - margin.right)/(length-1)
            vm.dim_slot = (slot<vm.min_axis_gap)?vm.min_axis_gap:slot
            // let width = margin.left + margin.right + (vm.columns.length-1) * vm.dim_slot
            let width = vm.axis[vm.columns[vm.columns.length-1]].ctn.x + margin.right
            let height = vm.app.layout.height
            vm.app.layout.width = width
            vm.app.renderer.resize(width,height)
            // vm.drawAllDataLine()
        },
        // 處理維度的 scale 
        d3ScaleSetting(extent){
            vm.scale = {}
            for(let column in extent){
                vm.scale[column] = {}
                let domain = extent[column]
                let range = domain[1] - domain[0]
                let gap = range / 4
                vm.scale[column]['scale'] = d3.scaleLinear().domain(domain).range([190,0])
                let ticks = [parseFloat(domain[0]).toFixed(4)]
                for(let i = 1; i < 4; i++){
                    ticks.push(parseFloat(domain[0]+i*gap).toFixed(4))
                }
                ticks.push(parseFloat(domain[1]).toFixed(4))
                ticks = new Set(ticks)
                vm.scale[column]['ticks'] = ticks
            }
        },
        // 軸線繪製內容
        drawAxis(){
            let margin = vm.app.layout.margin
            vm.columns.forEach((c,i) => {
                vm.axis[c] = {}
                // 軸線主體
                let axis = new PIXI.Container()
                // 欄位名稱顯示
                let label = vm.drawLabel(c,axis)
                // 軸線繪製主體
                let axisLine = vm.drawAxisLine(c,axis)
                // ticks 繪製
                let axisTicks = vm.drawAxisTicks(c,axis)

                let max_ticks_length = axisTicks.children[axisTicks.children.length-2].width
                
                axis.x = i * vm.dim_slot + margin.left+max_ticks_length

                // vm.axis[c] = axis
                vm.axis[c].ctn = axis
                axis.addChild(label)
                axis.addChild(axisLine)
                axis.addChild(axisTicks)
                vm.ctn_axis.addChild(axis)
            })
        },
        // 繪製軸線
        drawAxisLine(c,axis){
            // 单一 ctn_axisLine 包含两个 axisLine 以及 hitArea
            let ctn_axisLine = new PIXI.Container()
            let axisLine = new PIXI.Graphics()
            // let hitArea = new PIXI.Graphics()
            // lineStyle 控制圖形線條的（粗細，顏色，透明度）
            axisLine.lineStyle(1,0x000000,1)
            axisLine.begin
            axisLine.moveTo(0,0)
            axisLine.lineTo(0,vm.plot_height)
            axisLine.interactive = true
            axisLine.buttonmode = true

            axisLine.mousedown = vm.drawFilterStart
            axisLine.mousemove = vm.filterResize
            axisLine.mouseup = vm.drawFilterEnd
            axisLine.mouseout = vm.drawMouseOut
            axisLine.rightdown = vm.removeFilterBox

            axisLine.hitArea = new PIXI.Rectangle(axisLine.x-10, axisLine.y,20,190)
            axisLine.y = axis.label.y + axis.label.height + 5
            axisLine.label = c
            let filter_box = new PIXI.Container()
            axisLine.filter_box = filter_box
            // 儲存當前的 axisLine
            vm.axis[c].axisLine = axisLine
            vm.axis[c].filter_box = filter_box  

            ctn_axisLine.addChild(axisLine)
            ctn_axisLine.addChild(filter_box)
            return ctn_axisLine
        },
        // 繪製 Dimension 名稱
        drawLabel(column, axis){
			let label = new PIXI.Text(column
			, {fontFamily : vm.font, fontSize: vm.font_size, fill : 0x000000, align : 'center'})
			label.x = -vm.indicator_radius * 2
			label.rotation = - Math.PI / 6
            label.y = 50
            axis.label = label
            return label
        },
        // 繪製 ticks
        drawAxisTicks(column, axis){
            let scale = vm.scale[column].scale
            let ticks = vm.scale[column].ticks
            let axisTicks = new PIXI.Container()
            // let format1 = d3.format('.4s')
            let format2 = d3.format('.2f')
            ticks.forEach(tick => {
                let text = format2(tick)
                // if(tick > 1)
                //     text = format1(tick)
                // else
                //     text = format2(tick)
                let tick_label = new PIXI.Text(
                    text,
                    {
                        fontFamily : vm.font, 
                        fontSize: 12, 
                        fill : 0x000000, 
                        align : 'center'
                    }                       
                )
                tick_label.x = axis.label.x - tick_label.width - 10
                tick_label.y = vm.axis[column].axisLine.y + scale(tick) - tick_label.height/2

                let tick_line = new PIXI.Graphics()
                tick_line.lineStyle(1,0x000000,1)
                tick_line.moveTo(0,0)
                tick_line.lineTo(5,0)
                tick_line.x = vm.axis[column].axisLine.x - 5
                tick_line.y = tick_label.y + tick_label.height/2

                axisTicks.addChild(tick_label)
                axisTicks.addChild(tick_line)
            })
            axis.axisTicks = axisTicks
            return axisTicks
        },
        // 繪製選中資料點
        drawMaskDataLine(mask_pts){
            let latent_scatter = vm.eventBus.latent_scatter
            mask_pts.forEach(pt => {
                let data = latent_scatter.filterSelectedDimData(pt.data)
                if(pt.pcp === undefined){
                    vm.drawSingleLine(pt)      
                }
                else{
                    pt.pcp.alpha = vm.line_alpha
                }
            })
        },
        drawFilterDataLine(mask_pts){
            let latent_scatter = vm.eventBus.latent_scatter
            mask_pts.forEach(pt => {
                let data = latent_scatter.filterSelectedDimData(pt.data)
                if(pt.pcp === undefined){
                    vm.drawSingleLine(pt)      
                }
                else{
                    pt.pcp.alpha = vm.line_alpha
                }
                pt.pcp.filter = true 
            })            
        },
        // 绘制一条折线
        drawSingleLine(pt){
            let line = new PIXI.Graphics()
            let color_cb = vm.eventBus.latent_scatter.getColor
            let data = vm.eventBus.latent_scatter.filterSelectedDimData(pt.data)
            line.lineStyle(1, 0xffffff, 1)
            data.forEach((d,i) => {
                let c = vm.columns[i]
                let scale = vm.scale[c].scale
                let x = vm.axis[c].ctn.x
                let y = vm.axis[c].axisLine.y + scale(d)
                if( i === 0 ){
                    line.moveTo(x,y)
                }
                else{
                    line.lineTo(x,y)
                }
                line.tint = (color_cb)?color_cb(pt.x,pt.y):0x000000
                line.alpha = vm.line_alpha
            })
            line.pt = pt
            line.filter = false
            vm.ctn_lines.addChild(line)      
            pt.pcp = line  
        },
        // 移除當前的 data line
        removeLines(){
            vm.ctn_lines.children.forEach(line => {
                line.alpha = 0
                line.filter = false
            })
        },
        // 移除当前移动经过的 data line
        removeMoveLines(){
            if(!vm.eventBus){
                return 
            }
            let latent_scatter = vm.eventBus.latent_scatter
            // 存在有左鍵標記的 mask pts,且同時 pcp 的filter box 爲 0
            let filter_box = vm.columns.some(c => {
                return vm.axis[c].filter_box.children.length
            })
            if(latent_scatter.mask_group.length > 0 && filter_box){
                vm.ctn_lines.children.forEach(line => {
                    line.alpha = (line.pt.center_mark)?(line.filter)?vm.line_alpha:0:0
                })
            }
            else{
                vm.ctn_lines.children.forEach(line => {
                    line.alpha = (line.pt.center_mark || line.filter)?vm.line_alpha:0
                })
            }
        },
        // 刪除 filter box 
        removeFilterBox(e){
            let axisLine = e.currentTarget
            let label = axisLine.label
            let py = e.data.global.y - axisLine.y
            let box = axisLine.filter_box.children.filter(box => {
                let [min,max] = box.extent.map(y => {
                    return vm.scale[label].scale(y)
                })
                return max >= py && min <= py
            })[0]
            if(box !== undefined){
                axisLine.filter_box.removeChild(box)
                vm.removeLines()
                let [mask_pts] = vm.eventBus.latent_scatter.pcpFilter(vm.columns,vm.axis)
                vm.drawFilterDataLine(mask_pts)
            }
        },
        // 基本的 filter box 元件繪製
        initFilterBox(axis,box,x1,y1,width,height){
            box.clear()
            box.lineStyle(1,0x000000,1)
            box.beginFill(0xffffff)
            box.drawRect(x1,y1,width,height)
            box.endFill()
        },
        // 開始繪製 filter box
        drawFilterStart(e){
            vm.filter_start = true
            // filter start point
            vm.filter_sp = [e.data.global.x, e.data.global.y]
            let axis = e.currentTarget
            let box = new PIXI.Graphics()
            vm.initFilterBox(axis,box,0,0,0,0)
            axis.filter_box.addChild(box)
            e.axis = axis
        },
        // 改變 filter box 長度
        filterResize(e){
            if(vm.filter_start){
                // let axis_y = - e.currentTarget.y
                let y1 = vm.filter_sp[1]
                let y2 = e.data.global.y
                let axis = e.axis
                let current_box = axis.filter_box.children.slice(-1)[0]
                if( 0 > (y2-y1) ){
                    vm.initFilterBox(axis,current_box,axis.x-10,y2,20,y1-y2)
                    // 直接負數值繪製時，在開啓交互模式時會錯誤
                }
                else{
                    vm.initFilterBox(axis,current_box,axis.x-10,y1,20,y2-y1)
                }
                current_box.extent = [y1-axis.y,y2-axis.y]
            }
        },
        // 結束繪製
        drawFilterEnd(e){
            if(vm.filter_start){
                let axis = e.axis
                let current_box = axis.filter_box.children.slice(-1)[0]
                let label = axis.label
                if(current_box){
                    let scale = vm.scale[label].scale
                    let x1 = scale.invert(current_box.extent[0])
                    let x2 = scale.invert(current_box.extent[1])
                    current_box.extent = [Math.max.apply(null,[x1,x2]),Math.min.apply(null,[x1,x2])]
                }
                // filter box start position
                vm.filter_sp = undefined
                vm.filter_start = false
                vm.removeLines()
                let [mask_pts,cb] = vm.eventBus.latent_scatter.pcpFilter(vm.columns,vm.axis)
                vm.drawFilterDataLine(mask_pts)
            }
        },
        // mouseout
        drawMouseOut(e){
            vm.drawFilterEnd(e)
        },
        // 改變折線顏色
        changeTint(color_cb){
            vm.ctn_lines.children.forEach(line => {
                let pt = line.pt
                line.tint = color_cb(pt.curpos[0],pt.curpos[1])
            })            
        }
	},
	mounted() {
        vm = this
        d3 = vm.$d3
        PIXI = vm.$PIXI

        vm.axis = {}

        vm.pixiInit()
        vm.$refs.home.addEventListener('contextmenu', e => {e.preventDefault()})
	},
	beforeDestroy() {
        vm.ctn_lines.children.forEach(line => {
            line.pt.pcp = undefined
        })
		if (vm.app !== undefined) {
			vm.app.destroy()
		}
	}
}
</script>