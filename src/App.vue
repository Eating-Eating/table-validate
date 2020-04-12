<template>
  <div id="app">
      <el-main>
        <el-table
          border
          :data="tableList"
        >
          <el-table-column type="index" label="序号" width="50px"></el-table-column>
          <el-table-column label="姓名" prop="section_name"></el-table-column>
          <el-table-column label="最爱的食物">
            <template slot-scope="{ row }">
              {{row.grade_list | arrToStr}}
            </template>
          </el-table-column>
          <el-table-column label="操作">
            <template slot-scope="{ row }">
              <el-button @click="openDialog(row)">编辑</el-button>
            </template>
          </el-table-column>
        </el-table>
      </el-main>
      <el-dialog title="编辑老八最爱的食物" :visible.sync="editDialog" width="700px" 
        :before-close="handleBeforeClose">
        <el-button @click="addClass" class="addBtn">新增食物</el-button>
        
        <el-table
          border
          :data="myTable"
        >
          <el-table-column type="index" label="序号"></el-table-column>
          <el-table-column prop="section_name" label="姓名"></el-table-column>
          <el-table-column label="最爱的食物">
            <template slot-scope="{ row,$index }">
              <div class="el-form-item" :class="{
                  'is-error': errorObj.hasOwnProperty($index)
                }">
                <div class="el-form-item__content">
                  <el-input v-model="row.grade_name" maxlength="10"></el-input>
                  <transition name="el-zoom-in-top">
                    <div
                      v-if="errorObj.hasOwnProperty($index)"
                      class="el-form-item__error"
                    >
                      {{errorObj[$index]}}
                    </div>
                  </transition>
                </div>
              </div>
            </template>
          </el-table-column>
          <el-table-column label="操作">
            <template slot-scope="{ row,$index }">
              <el-button @click="delClass(row,$index)">删除</el-button>
            </template>
          </el-table-column>
        </el-table>
        <span slot="footer" class="dialog-footer">
          <el-button @click="handleBeforeClose">取 消</el-button>
          <el-button type="primary" @click="putGrades" v-loading="saveLoading">保 存</el-button>
        </span>
      </el-dialog>
  </div>
</template>

<script>
let originMap = new Map()
export default {
  name: 'app',
    data() {
      return {
        tableList: [],
        editDialog:false,
        innerVisible:false,
        tempSectionID:Number,
        myTable:[],
        tempRow:{},
        //验证类
        validateState:'error',
        errorObj:{},
        saveLoading:false
      }
    },
    filters: {
      arrToStr: function(value) {
        return value.join('、')
      }
    },
    methods: {
      openDialog(row) {
        this.editDialog = true
        this.tempRow = row
        this.myTable = row.grade_list.map((key,index) => ({
          id:index,
          section_name:row.section_name,
          grade_name:key
        }))
        
        this.myTable.forEach(key =>{
          originMap.set(key.grade_name,key.id)
        })
      },
      handleBeforeClose(done){
      this.$confirm("编辑的内容尚未保存，确定关闭弹窗吗？")
        .then(() => {
          this.tempRow = {}
          this.myTable = []
          originMap = new Map()
          this.errorObj = {}
          console.log('tag', '')
          try {
            done();
            
          } catch (error) {
            this.editDialog = false
          }
        })
        .catch(() => {});
      },
      addClass(){
        this.myTable.push({
          section_name:this.tempRow.section_name,
          grade_name:''
        })
      },
      delClass(asd,$index){
        this.myTable.splice($index,1)
      },
      putGrades(){
        //去空
        this.myTable = this.myTable.filter(key=> key.grade_name.trim() !== '')
        // 验证重复
        let map = new Map()
        this.errorObj = {}
        this.myTable.forEach((key,index)=>{
          if(map.has(key.grade_name)){
            let oldVal = map.get(key.grade_name)
            map.set(key.grade_name,[...oldVal,index])
            return
          }
          map.set(key.grade_name,[index])
        })
        console.log('map', map)
        map.forEach((value)=>{
          if(value.length > 1){
            value.forEach(key=> { 
              //存入重复项以及验证错误的原因
              this.$set(this.errorObj,key,'老八不喜欢重复的食物！')
            })
          }
        })
        //有重复阻止发送请求
        if(Object.keys(this.errorObj).length !== 0) return
        //修改痕迹
        //第1步，判断新旧是否有重名，有重名的先直接去掉,属于不变的组
        let tempNew = this.myTable.filter((key) => {
          if(originMap.has(key.grade_name)) {
            originMap.delete(key.grade_name)
            return false
          }
          return true
        })
        //第2.1步，判断有id的名字是否一致，不一致属于修改组
        //第2.2步，判断新旧带id的数量是否一致，origin少了的部分为删除组
        //第3步，剩下的都是新增组
        //反转originMap中键值对
        
        let edited = new Map()
        let noIDGrade = []
        let idMap = new Map()
        originMap.forEach((value,key)=>{
          idMap.set(value,key)
        })
        tempNew.forEach(key=>{
          if(key.id !== undefined) {
            edited.set(idMap.get(key.id),key.grade_name)
            idMap.delete(key.id)
          }else{
            noIDGrade.push(key.grade_name)
          }
        })
        const h = this.$createElement;
        let renderEdit = []
        let renderAdd = []
        let renderDel = []
        edited.forEach((value,key)=>{
          renderEdit.push(h('p',`${key}修改为${value}`))
        })
        noIDGrade.forEach((value)=>{
          renderDel.push(h('p',`新增：${value}`))
        })
        idMap.forEach((value)=>{
          renderAdd.push(h('p',`已删除：${value}`))
        })
        //判断修改组、新增组、删除组是否为空
        if(renderEdit.length === 0 && renderAdd.length === 0 && renderDel.length === 0) return
        this.$msgbox({
          title: "修改确认",
          message: h("div", null, [
            h( 
              "p",renderEdit
            ),
            h(
              "p",renderAdd
            ),
            h(
              "p",renderDel
            ),
          ]),
          showCancelButton: true,
          showClose: false,
          confirmButtonText: "确认",
          cancelButtonText: "取消"
        })
          .then(() => {
            this.tableList[0].grade_list = []
            this.myTable.forEach(key=>{
              this.tableList[0].grade_list.push(key.grade_name)
            })
            
          this.tempRow = {}
          this.myTable = []
          originMap = new Map()
          this.errorObj = {}
          this.editDialog = false
          })
          .catch(() => {});
      },
    },
    created(){
      this.tableList = [{
        section_name: "老八",
        grade_list: ["臭豆腐", "俘虏", "臭撸虾", "嘎嘣脆嗲", "胡罗贝", "柠檬"]
      }]
    }
  
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
