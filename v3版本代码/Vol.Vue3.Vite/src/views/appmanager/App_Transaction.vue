<!--
*Author：jxx
 *Contact：283591387@qq.com
 *代码由框架生成,任何更改都可能导致被代码生成器覆盖
 *业务请在@/extension/appmanager/App_Transaction.js此处编写
 -->
<template>
    <view-grid ref="grid"
               :columns="columns"
               :detail="detail"
               :editFormFields="editFormFields"
               :editFormOptions="editFormOptions"
               :searchFormFields="searchFormFields"
               :searchFormOptions="searchFormOptions"
               :table="table"
               :extend="extend">
    </view-grid>
</template>
<script>
    import extend from "@/extension/appmanager/App_Transaction.jsx";
    import { ref, defineComponent } from "vue";
    export default defineComponent({
        setup() {
            const table = ref({
                key: 'Id',
                footer: "Foots",
                cnName: '导入导出',
                name: 'App_Transaction',
                url: "/App_Transaction/",
                sortName: "Id"
            });
            const editFormFields = ref({"Name":"","TransactionType":"","CowType":"","PhoneNo":"","Describe":""});
            const editFormOptions = ref([[{"title":"姓名","required":true,"field":"Name"}],
                              [{"dataKey":"enable","data":[],"title":"是否可用","required":true,"field":"TransactionType","type":"select"}],
                              [{"dataKey":"nav","data":[],"title":"购买类型","field":"CowType","type":"select"}],
                              [{"title":"电话","required":true,"field":"PhoneNo","type":"phone"}],
                              [{"title":"描述","required":true,"field":"Describe","type":"textarea"}]]);
            const searchFormFields = ref({"Name":"","PhoneNo":"","TransactionType":"","CowType":[],"Creator":"","CreateDate":""});
            const searchFormOptions = ref([[{"title":"姓名","field":"Name","type":"text"},{"title":"电话","field":"PhoneNo","type":"text"},{"dataKey":"enable","data":[],"title":"是否可用","field":"TransactionType","type":"select"}],[{"dataKey":"nav","data":[],"title":"购买类型","field":"CowType","type":"selectList"},{"title":"提交人","field":"Creator"},{"title":"提交时间","field":"CreateDate","type":"datetime"}]]);
            const columns = ref([{field:'Id',title:'主键ID',type:'int',width:90,hidden:true,readonly:true,require:true,align:'left'},
                       {field:'Name',title:'姓名',type:'string',width:120,require:true,align:'left',sort:true},
                       {field:'PhoneNo',title:'电话',type:'string',link:true,width:150,require:true,align:'left'},
                       {field:'Quantity',title:'数量',type:'int',width:90,require:true,align:'left'},
                       {field:'TransactionType',title:'是否可用',type:'int',bind:{ key:'enable',data:[]},width:120,require:true,align:'left'},
                       {field:'CowType',title:'购买类型',type:'string',bind:{ key:'nav',data:[]},width:90,align:'left'},
                       {field:'Describe',title:'描述',type:'string',width:190,require:true,align:'left'},
                       {field:'Enable',title:'是否启用',type:'byte',width:90,hidden:true,align:'left'},
                       {field:'CreateID',title:'创建人Id',type:'int',width:90,hidden:true,align:'left'},
                       {field:'Creator',title:'提交人',type:'string',width:130,align:'left'},
                       {field:'CreateDate',title:'提交时间',type:'datetime',width:150,align:'left',sort:true},
                       {field:'ModifyID',title:'修改人ID',type:'int',width:90,hidden:true,align:'left'},
                       {field:'Modifier',title:'修改人',type:'string',width:130,hidden:true,align:'left'},
                       {field:'ModifyDate',title:'修改时间',type:'datetime',width:150,hidden:true,align:'left',sort:true}]);
            const detail = ref({
                cnName: "#detailCnName",
                table: "#detailTable",
                columns: [],
                sortName: "",
                key: ""
            });
            return {
                table,
                extend,
                editFormFields,
                editFormOptions,
                searchFormFields,
                searchFormOptions,
                columns,
                detail,
            };
        },
    });
</script>
