<template>
    <div class="mod-config">
        <el-form :inline="true" :model="dataForm" @keyup.enter.native="getDataList()">
            <el-form-item>
                <el-input v-model="dataForm.key" placeholder="参数名" clearable></el-input>
            </el-form-item>
            <el-form-item>
                <el-button @click="getDataList()">查询</el-button>
            </el-form-item>
        </el-form>
        <el-table
                :data="dataList"
                border
                v-loading="dataListLoading"
                @selection-change="selectionChangeHandle"
                style="width: 100%;">
            <el-table-column
                    header-align="center"
                    align="center"
                    label="状态">
                <template slot-scope="scope">
                    <el-popover v-if="scope.row.checkName" trigger="hover" placement="top" @show="getReason(scope.row)">
                        <div v-loading="reasonLoading">
                            <p>操作人: {{ scope.row.checkName }}</p>
                            <p>驳回原因: {{ scope.row.reason }}</p>
                        </div>
                        <div slot="reference">
                            <el-tag type="danger" size="medium">被驳回</el-tag>
                        </div>
                    </el-popover>
                    <el-tag type="success" v-else>未提交</el-tag>
                </template>
            </el-table-column>
            <purchase ref="purchase"></purchase>
            <el-table-column
                    fixed="right"
                    header-align="center"
                    align="center"
                    width="150"
                    label="操作">
                <template slot-scope="scope">
                    <el-button type="text" size="small" @click="subHandle(scope.row.processInstanceId)">提交
                    </el-button>
                    <el-button type="text" size="small" @click="addOrUpdateHandle(scope.row.purchaseId)">修改
                    </el-button>
                    <el-button type="text" size="small" @click="delHandle(scope.row.purchaseId)">删除
                    </el-button>
                    <el-button type="text" size="small" @click="detHandle(scope.row.purchaseId)">查看详情
                    </el-button>
                </template>
            </el-table-column>
        </el-table>

        <el-pagination
                @size-change="sizeChangeHandle"
                @current-change="currentChangeHandle"
                :current-page="pageIndex"
                :page-sizes="[10, 20, 50, 100]"
                :page-size="pageSize"
                :total="totalPage"
                layout="total, sizes, prev, pager, next, jumper">
        </el-pagination>
        <pDetailed v-if="pDetailedVisible" ref="pDetailed" @refreshDataList="getDataList"></pDetailed>
        <!-- 弹窗, 新增 / 修改 -->
        <add-or-update v-if="addOrUpdateVisible" ref="add" @refreshDataList="getDataList"></add-or-update>
    </div>
</template>

<script>
    import add from './add'
    import purchase from './purchase'
    import pDetailed from "./pDetailed";

    export default {
        data() {
            return {
                dataForm: {
                    key: ''
                },
                dataList: [],
                pageIndex: 1,
                pageSize: 10,
                totalPage: 0,
                pDetailedVisible: false,
                dataListLoading: false,
                dataListSelections: [],
                addOrUpdateVisible: false,
                reasonLoading: false
            }
        },
        components: {
            purchase,
            pDetailed,
            add
        },
        activated() {
            this.getDataList()
        },
        methods: {
            // 获取数据列表
            getDataList() {
                this.dataListLoading = true
                this.$http({
                    url: this.$http.adornUrl('/drugs_purchase/purchase/list'),
                    method: 'get',
                    params: this.$http.adornParams({
                        'page': this.pageIndex,
                        'limit': this.pageSize,
                        'purchaseId': this.dataForm.key
                    })
                }).then(({data}) => {
                    if (data && data.code === 200) {
                        this.dataList = data.page.list
                        this.totalPage = data.page.totalCount
                    } else {
                        this.dataList = []
                        this.totalPage = 0
                    }
                    this.dataListLoading = false
                })
            },
            detHandle(pid) {
                this.pDetailedVisible = true
                this.$nextTick(() => {
                    this.$refs.pDetailed.init(pid)
                })
            },
            //获取复选框
            handleSelectionChange(val) {
                this.multipleSelection = val;
            },
            // 每页数
            sizeChangeHandle(val) {
                this.pageSize = val
                this.pageIndex = 1
                this.getDataList()
            },
            // 当前页
            currentChangeHandle(val) {
                this.pageIndex = val
                this.getDataList()
            },
            // 多选
            selectionChangeHandle(val) {
                this.dataListSelections = val
            },
            // 新增 / 修改
            addOrUpdateHandle(id) {
                this.$router.push({path:'/drugs_purchase/update/'+id})
            },
            //获取药
            getData() {
                this.dataListLoading = true
                this.$http({
                    url: this.$http.adornUrl('/drugs/detailed/list'),
                    method: 'get',
                    params: this.$http.adornParams({})
                }).then(({data}) => {
                    if (data && data.code === 200) {
                        this.tableData = data.page.list
                        } else {
                        this.dataList = []
                    }
                    this.dataListLoading = false
                })
            },
            subHandle(id) {
                var ids = id ? [id] : this.dataListSelections.map(item => {
                    return item.processInstanceId
                })
                this.$confirm(`确定对这${ids.length}条数据进行[${id ? '提交' : '批量提交'}]操作?`, '提示', {
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    type: 'warning'
                }).then(() => {
                    this.$http({
                        url: this.$http.adornUrl('/drugs_purchase/purchase/subOrder'),
                        method: 'post',
                        data: this.$http.adornData(ids,false)
                    }).then(({data}) => {
                        if (data && data.code === 200) {
                            this.$message.success("操作成功")
                            this.getDataList()
                        } else {
                            this.$message.error(data.msg)
                        }
                    })
                })
            },
            getReason(row){
                if (row.reason) return
                this.reasonLoading = true
                this.$http({
                    url: this.$http.adornUrl('/activiti/reason'),
                    method: 'post',
                    params: this.$http.adornParams({'processInstanceId':row.processInstanceId})
                }).then(({data}) => {
                    if (data && data.code === 200) {
                        this.$set(row, 'reason', data.reason)
                        this.reasonLoading = false
                    } else {
                        this.$message.error(data.msg)
                    }
                })
            }
        }
    }
</script>
