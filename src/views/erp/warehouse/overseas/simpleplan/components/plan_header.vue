<template>
	<div class="con-header">
	<el-row>
		<el-space>
			<el-radio-group v-model="queryParam.auditstatus" @change="handleQuery" >
			        <el-radio-button label=""  value=""  >全部 </el-radio-button >
			        <el-radio-button label="1" value="1" >备货中</el-radio-button >
			        <el-radio-button label="2" value="2" >已完成</el-radio-button >
			</el-radio-group>
			<Warehouse
			@changeware="gettoWarehouse" 
			 warehouseType="oversea_usable"
			 defaultValue="only" 
			:isform="true" />
			<Datepicker ref="datepickers"   @changedate="changedate" />
			<el-input  v-model="searchKeywords" placeholder="输入订单编码,SKU或名称" @input="handleQuery" clearable class="input-with-select" >
			   <template #append>
			     <el-button @click="handleQuery">
			        <el-icon style="font-size: 16px;align-itmes:center">
			         <search />
			      </el-icon>
			     </el-button>
			   </template>
			 </el-input>
		</el-space>
		<div class='rt-btn-group'>
			 <el-switch
			    v-model="queryParam.isdelete"
			    class="ml-2"
				@change="handleQuery"
			    inline-prompt
				size="large"
			    active-text="已删除"
			    inactive-text="未删除"
			  />
		</div>
			
	</el-row>
			<el-row>
				<el-space>
					<el-button type="primary" @click="handePlan">新增备货</el-button> 
				 
					    <el-dropdown split-button  type="success" @click="handleAdd">
					      <span>完成备货</span>
					      <template #dropdown>
					        <el-dropdown-menu>
					          <el-dropdown-item  @click="handleAddOut">完成备货并出库</el-dropdown-item>
					        </el-dropdown-menu>
					      </template>
					    </el-dropdown>
					 <el-button type="danger"  @click="deletes">删除</el-button> 
				</el-space>
				<div class='rt-btn-group'>
					<el-button class='ic-btn' title='帮助文档'>
						<help theme="outline" size="16" :strokeWidth="3" />
					</el-button>
				</div>
			</el-row>
 </div>
			<el-dialog
			   v-model="uploadVisible"
			   title="导入调库单"
			   width="400px"
			 >
			 <el-upload
			    :drag="true"
			    action
			    :http-request="uploadFiles"
			    :limit="1"
			    :before-upload="beforeUpload" 
			    :show-file-list="true" 
			    :headers="headers" 
			    accept=".xls,.xlsx"
			    multiple
			   >
			     <el-icon class="font-large"><upload-filled /></el-icon>
			     <div class="el-upload__text">
			      拖拽文件到此处或 <em>点击上传</em>
			     </div>
			   </el-upload>
			 <template #footer>
			   <span class="dialog-footer">
				   <div class="flex-center-between">
				 <el-button type="success" @click.stop="downloadTemp" plain>下载模板</el-button>
				 <div>
			     <el-button @click="uploadVisible = false">取消</el-button>
			     <el-button type="primary" @click.stop="uploadExcel">
			       上传文件
			     </el-button></div></div>
			   </span>
			 </template>
			  </el-dialog>
 <PlanCreate      ref="planCreateRef"      @change="handleQuery"></PlanCreate>
</template>

<script setup>
	import {ref,reactive,onMounted,watch,h,toRefs} from 'vue'
	import {Help,Plus,MenuUnfold,SettingTwo} from '@icon-park/vue-next';
	import {ElMessage,ElDivider} from 'element-plus';
	import {Search,ArrowDown,UploadFilled,House} from '@element-plus/icons-vue';
    import Warehouse from '@/components/header/warehouse.vue';
	import Category from '@/components/header/category.vue';
	import Datepicker from '@/components/header/datepicker.vue';
	import { useRouter } from 'vue-router';
	import dispatchApi from '@/api/erp/inventory/dispatchApi.js';
	import planFormApi from "@/api/erp/order/planFormApi.js";
	import Platform from './platform.vue';
	import orderApi from "@/api/erp/order/orderApi.js";
	import PlanCreate from "@/views/erp/warehouse/overseas/inv/components/plan_create_dialog.vue";
	const emit = defineEmits(['getdata']);
	const createOrderRef=ref();
    const planCreateRef=ref();
    const state =reactive({
		uploadVisible:false,
		queryParam:{
			search:'',
			auditstatus:'',
			platformid:''
		},
		selectRows:[],
		myfile:null,
		searchKeywords:'',
		warehouse:{},
	})
	const {
		orderStateList,
		uploadVisible,
		queryParam,
		activeStatus,
		myfile,
		selectRows,
		searchKeywords
	}=toRefs(state)
	function handleAdd(){
		var ids=[];
		  state.selectRows.forEach(item=>{
			  ids.push(item.id);
		  })
		  planFormApi.done(ids).then((res)=>{
		  	ElMessage.success("操作成功");
		  	handleQuery();
		  })
	}
	function handleAddOut(){
		var ids=[];
		  state.selectRows.forEach(item=>{
			  ids.push(item.id);
		  })
		  planFormApi.doneOut(ids).then((res)=>{
		  	ElMessage.success("操作成功");
		  	handleQuery();
		  })
	}
	
	//导入
	function upload(){
		state.uploadVisible = true;
	}
	//日期改变
	function changedate(dates){
		state.queryParam.fromDate=dates.start;
		state.queryParam.toDate=dates.end;
		handleQuery();
	}
	function gettoWarehouse(warehouseid,type,warehouse){
		state.queryParam.warehouseid=warehouseid;
		state.warehouse=warehouse;
		emit('getdata',state.queryParam);
	}
	function handlePlatform(value){
		state.queryParam.platformid=value;
		emit('getdata',state.queryParam);
	}
	function handleQuery(){
		state.queryParam.sku=state.searchKeywords;
		state.selectRows=[];
		emit('getdata',state.queryParam);
	}
	function selectChange(rows){
		state.selectRows=rows;
	}
	//文件上传之前
	function beforeUpload(file){
		if (file.type != "" || file.type != null || file.type != undefined){
		    //截取文件的后缀，判断文件类型
			const FileExt = file.name.replace(/.+\./, "").toLowerCase();
			//计算文件的大小
			const isLt5M = file.size / 1024  < 5000; //这里做文件大小限制
			//如果大于50M
			if (!isLt5M) {
				ElMessage.error(  '上传文件大小不能超过 5MB!!');
				return false;
			}
			else {
			   return true;
			}
		}
	}
	function handePlan(){
		var param=JSON.parse(JSON.stringify(state.queryParam));
		param.ischeck=true;
		param.warehousename=state.warehouse.name;
		planCreateRef.value.show([],param);
	}
	function uploadFiles(item){
		//上传文件的需要formdata类型;所以要转
		state.myfile=item.file;
	}
	function uploadExcel(){
		let FormDatas = new FormData();
		FormDatas.append('file',state.myfile);
		dispatchApi.uploadExcel(FormDatas).then(function(res){
		 
		})
	}
	function downloadTemp(){
		dispatchApi.downExcelTemp();
	}
	function deletes(){
		var ids=[];
		  state.selectRows.forEach(item=>{
			  ids.push(item.id);
		  })
		planFormApi.remove(ids).then((res)=>{
			ElMessage.success("删除成功");
			globalTable.value.loadTable(state.queryParams);
		})
	}
	onMounted(()=>{
	 
	});
	
	defineExpose({
		selectChange,
	})
</script>

<style scoped="scoped">
	.text-orange{
		font-weight: 700;
		color:var(--el-color-primary);
		font-size: 12px;
	}
	.font-48{
		font-size: 48px;
		    color: #999;
	}
</style>
