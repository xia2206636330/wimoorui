<template>
	<div class="saledialog">
	<el-dialog v-model="saleVisable" :title="productData.sku" @close="handleClose" :modal="false" :lock-scroll="false"    draggable>
		  <template #header>
		    <div style="padding-right: 20px; padding-top: 4px;">
				<el-tabs
				   v-model="activeMarket"
				   class="markets-tabs-dialog"
				   @tab-change="handleClick"
				   
				 >
				  <el-tab-pane  v-for = "(item,index) in marketList" :label="item" :name="item"  >
					  
				  </el-tab-pane>
				 </el-tabs>
			 </div>
		  </template>
			<div class="flex-center">
						 <el-radio-group v-model="radiotime" @change="changeTimeType" size="small">
						       <el-radio-button label="30天" />
						       <el-radio-button label="60天" />
						       <el-radio-button label="90天" />
						     </el-radio-group>
			</div>
		<div id='salechart' style='height:160px;width:100%' >
		   
		</div>
		
	</el-dialog>
	</div>
</template>

<script setup>
	import * as echarts from 'echarts'
	import { ElDivider } from 'element-plus'
	import {ref,reactive,onMounted,watch,h,nextTick} from 'vue';
	import marketApi from '@/api/amazon/market/marketApi.js';
	import orderApi from '@/api/erp/order/orderApi.js';
	import orderPlanApi from '@/api/erp/order/orderPlanApi.js';
		let activeMarket = ref();
		let saleVisable = ref(false)
		let ordercheck=ref(false);
		let productData = reactive({})
		let radiotime =ref("30天")
		let spacer = h(ElDivider, { direction: 'vertical' })
		let marketList=ref([]);
		let chart;
		let chartorder;
		let linetype=ref(['销量']);
		let skuname=ref("");
		let totalsales=ref(0);
		let totalorder=ref(0);
		var myChart = null;
		function handleClose(){
			 if(myChart!=null){
				 myChart.dispose();
			 }
		}
	    function saleChart(){
				if(myChart!=null){
					 myChart.clear();
				}
				myChart = echarts.init(document.getElementById('salechart'));
				var myseries=[];
				totalsales.value=0;
				totalorder.value=0;
				var legends=[];
				if(chartorder){
					chartorder.lines.forEach((line,index)=>{
						skuname.value=line.name;
						legends.push(line.name);
						totalorder.value=totalorder.value+line.summary;
						myseries.push({
										smooth: 0.8,
										type: 'line',
										data: line.data,
										name:line.name,
										lineStyle:{
											color:'#409eff',
										},
										itemStyle:{
											color:'#409eff',
										},
										label:{
											show:false,
										},
					                   })
					}) 
				}
				
				if(chart){
					chart.lines.forEach((line,index)=>{
						skuname.value=line.name;
						legends.push(line.name);
						totalsales.value=totalsales.value+line.summary;
						myseries.push({
										smooth: 0.5,
										type: 'line',
										data: line.data,
										name:line.name,
										lineStyle:{
											color:'#ffa400',
										},
										itemStyle:{
											color:'#ffa400',
										},
										label:{
											show:true,
										},
										showAllSymbol:radiotime.value=="30天"?true:false,
										markLine:{
														data : [ {
															silent:false,  
															type : 'average',
															name : '平均值'
														} ],
														itemStyle : {
															normal : {
																color : '#ffa400',
																lineStyle : {
																	color : '#ffa400'
																},
																label : {
																	show : true,
																	textStyle : {
																		color :'#ffa400',
																		fontWeight : 'bold',
																	}
																},
															},
														}
													},
					                   })
					}) 
				}
			
				var option = {
					visualMap : {
								show: false,
								dimension: 0,
								pieces: [],  //pieces的值由动态数据决定
								outOfRange: {
							    				color: '#ffa400'
							    			},
								inRange:{                               //定义 在选中范围中 的视觉元素
									color: '#f1f2f4'
								}
							},
				     tooltip : { trigger : 'axis', },
				     legend:{
					  type: "plain",
					  icon: "circle",
					  itemWidth: 8,
					  selected: {
								legendsfirst: true
							},
					  selectedMode:"single"
				     },  
				      xAxis: {
								boundaryGap:false,
								data: chart?chart.labels:chartorder.labels,
								axisLine:{   show: false },
								axisTick:{ show: false },
								axisLabel:{  color:"#acb0b9" },
							},
							grid:{
								right:32,
								left:32,
				                 bottom:20,
								 top:28,
							},
				      yAxis: {
								axisLabel:{
				                    color:"#acb0b9"
								},
								splitLine:{
									lineStyle:{
									color:"rgba(170,170,170,0.2)"
									}
								}
							},
				      series: myseries
				}
				 initVisualMapPieces(option,chart);//此处使用分段红色线条表示连续无货，代替markpoint
				 myChart.setOption(option,true);
				   var timer=setTimeout(()=>{myChart.resize();
				               clearTimeout(timer);
				              },1000);
				   
				 window.addEventListener('resize',()=>{
						   myChart.resize();
				 })
			}
			function initVisualMapPieces(option,chart){
				chart.lines.forEach(line=>{
					if(line.markpoint.length>0){
						var labelsmap={};
							for(var m in line.markpoint){
								if(chart.labels.includes(line.markpoint[m].xaxis)){
									labelsmap[line.markpoint[m].xaxis]=1;
								}
							}
						var p = 0;
						for(var i = 0; i < chart.labels.length; i++){
							if(labelsmap[chart.labels[i]]==1&&labelsmap[chart.labels[i+1]]==1) {
								option.visualMap.pieces[p] = {gte:i,lt:i+1,color:"#EB6A79"}; 
								p++; 
							}
						}
					}
				})
				if(option.visualMap.pieces.length==0){
					 option.visualMap.inRange.color='#ffa400';
				}
			}
			function handleClick(){
				if(productData.warehouseid!=activeMarket.value){
					productData.warehouseid=activeMarket.value;
					handleChartSaleLine();
				}
			}
			function handleOrderLine(value){
				if(value==true){
					linetype.value=['销量','订单'];
				}else{
					linetype.value=['销量'];
				}
				handleChartSaleLine();
			}
			async function handleChartSaleLine(){
					await orderApi.salesLine(productData).then((res)=>{
						if(res.data){
							 chart=res.data;
						}
					});
				nextTick(()=>{
					saleChart();
				});
			}
			function show(row){
				var param={"sku":row.sku, "amount":row.amount};
				saleVisable.value=true;
				radiotime.value="30天";
				linetype.value=['销量'];
				ordercheck.value=false;
				productData.daysize=30;
				productData.sku=row.sku;
				productData.warehouseid=null;
				nextTick(()=>{
					var list=[]
					orderPlanApi.getExpandCountryData(param).then(res=>{
						if(res.data){
							res.data.forEach(item=>{
								if(!list.includes(item.mycountry)){
									list.push(item.mycountry);
								}
							})
							activeMarket.value=list[0];
							productData.warehouseid=list[0];
							 
						}
						marketList.value=list;
						handleChartSaleLine();
					});
				});
			}
			defineExpose({
			  show
			}) 
			function changeTimeType(){
				if(radiotime.value=="30天"){
					productData.daysize=30;
				}else if(radiotime.value=="60天"){
					productData.daysize=60;
				}else if(radiotime.value=="90天"){
					productData.daysize=90;
				}
				handleChartSaleLine();
			}
			 
</script>

<style >
	.floatcenter{
		 padding-top:80px;
	}
	 .mybtn-radio .el-checkbox-button__inner{
		 padding: 5px 11px;
		 font-size:12px;
	 }
	.saledialog .el-overlay-dialog{
		text-align: right;
	}
	.saledialog .el-dialog{
/* 		margin: inherit;
		display: inline-block; */
		text-align: left;
		box-shadow: var(--el-box-shadow-light)
	}
	.saledialog .flex-center{
		display: flex;
		justify-content: space-between;
	}
	.markets-tabs-dialog .el-tabs__header{
		margin:0;
	}
	.saledialog .el-dialog__header,.saledialog .el-dialog__body{
		padding-top:5px !important;
	} 
</style>

