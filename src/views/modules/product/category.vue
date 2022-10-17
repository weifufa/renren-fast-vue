<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId"
      :default-expanded-keys="expandedkey" :draggable="draggable" :allow-drop="allowDrop" @node-drop="handleDrop"
      ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
            Append
          </el-button>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
          <el-button type="text" size="mini" @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="活动名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';
export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      pCid: [],
      draggable: false,
      updatedNodes: [],
      maxLevel: 0,
      title: "",
      dialogType: "", //edit,add
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null,
      },
      dialogVisible: false,
      menus: [],
      expandedkey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据....", data.data);
        this.menus = data.data;
      });
    },
    batchDelete() {
      let catIds = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      let pCid=[];
      console.log("被选中的元素", checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds[i] = checkedNodes[i].catId;
        pCid[i]= checkedNodes[i].parentCid;
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            message: "菜单批量删除成功！",
            type: "success",
          });
          this.getMenus();
          //设置需要默认展开的菜单
         this.expandedkey =pCid;
        });
      }).catch(() => { });
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updatedNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功！",
          type: "success",
        });
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
        this.pCid = [];
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      this.updatedNodes = [];//初始置空
      console.log("tree drop: ", draggingNode, dropNode, dropType);
      //1.当前节点最新的父结点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropNode == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId; //取出当前父结点
        siblings = dropNode.parent.childNodes; //
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      this.pCid.push(pCid);
      //2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的结点id
          let catLevel = draggingNode.level;
          //需要改层级，还有子节点层级
          if (siblings[i].level != draggingNode.level) {
            //当前结点的层级发送变化
            catLevel = siblings[i].level; //赋值最新层级
            //修改他子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          //需要改父id
          this.updatedNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updatedNodes.push({ catId: siblings[i].data.catId, sort: i });
        }

      }
      //3.当前拖拽节点的最新层级

      console.log("updateNode", this.updatedNodes);


    },
    updateChildNodeLevel(node) {
      //修改当前结点的所有子节点层级
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data; //拿到当前节点的数据
          this.updatedNodes.push({
            catId: cNode.catId, //获取子节点的id
            catLevel: node.childNodes[i].level,//更新子节点的层级
          });
          this.updateChildNodeLevel(node.childNodes[i]); //因为结点还有子节点，所以递归
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // //初始化深度置为1;
      this.maxLevel = 0;
      //1.被拖动的当前节点以及所在的父结点总层数不能大于3
      //1)、被拖动的当前结点总层数
      // console.log("allowDrop", draggingNode, dropNode, type);
      this.countNodeLeve(draggingNode);
      //当前正在拖动的结点+父结点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level + 1);
      // console.log("当前位置层级：", draggingNode.data.catLevel);
      // console.log("递归出来的最大深度：", this.maxLevel);
      // console.log("当前结点深度：", deep);
      // console.log("目标深度", dropNode.level);

      if (type == "inner") {
        //如果是拖到深度里边那就是相加关系
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3; // 前边和后边只用判断当前结点深度加上父结点深度是否大于3
      }
    },
    countNodeLeve(node) {
      // this.maxLevel=node.catLevel;//初始化当前最大深度为当前深度
      //找到所有子节点，找到最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLeve(node.childNodes[i]);
        }
      }
    },
    edit(data) {
      this.dialogType = "edit";
      this.title = "修改分类";
      console.log("要修改的数据", data);
      this.dialogVisible = true;
      //发送请求获取当前结点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("要回显的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    append(data) {
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //转换数字再+1
      this.category.catId = null;
      this.category.name = null;
      this.category.icon = "";
      this.category.productUnit = "";
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //修改三级分类
    editCategory() {
      var { catId, name, icon, productUnit } = this.category; //解构

      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功！",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = [this.category.parentCid];
      });
    },
    //添加三级分类
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功！",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = [this.category.parentCid];
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除当前【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功！",
              type: "success",
            });
            //刷新出新的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedkey = [node.parent.data.catId];
          });
        })
        .catch(() => { });
      console.log("remove", node, data);
    },
  },
  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() { },
  beforeCreate() { }, //生命周期 - 创建之前
  beforeMount() { }, //生命周期 - 挂载之前
  beforeUpdate() { }, //生命周期 - 更新之前
  updated() { }, //生命周期 - 更新之后
  beforeDestroy() { }, //生命周期 - 销毁之前
  destroyed() { }, //生命周期 - 销毁完成
  activated() { }, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style scoped>

</style>