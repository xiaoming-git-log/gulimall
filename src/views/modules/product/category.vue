<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="关闭拖拽"
      inactive-text="开启拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" type="primary" @click="submitDraggable"
      >提交拖拽结果</el-button
    >
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      @node-click="nodeClick"
      show-checkbox
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="handleCategory">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件(比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      curCatPidListExpand: [],
      updateNodes: [],
      curCatPid: "",
      draggable: false,
      maxLevel: 0,
      isEdit: "",
      dialogTitle: "",
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      dialogVisible: false,
      category: {
        name: "",
        parentCid: "",
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        catId: null,
      },
    };
  },
  methods: {
    //批量删除功能
    batchDelete() {
      let treeNodes = this.$refs.menuTree.getCheckedNodes();
      let catIds = [];
      let catNames = [];
      if (treeNodes && treeNodes.length > 0) {
        for (let i = 0; i < treeNodes.length; i++) {
          catIds.push(treeNodes[i].catId);
          catNames.push(treeNodes[i].name);
        }
        this.$confirm(`是否批量删除当前【${catNames}】菜单?`, "提示", {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        })
          .then(() => {
            this.$http({
              url: this.$http.adornUrl("/product/category/delete"),
              method: "post",
              data: this.$http.adornData(catIds, false),
            }).then(({ data }) => {
              this.getMenus();
              this.$message.success("批量删除菜单成功");
            });
          })
          .catch(() => {
            this.$message.info("已取消删除")
          });
      }
    },
    //提交拖拽结果
    submitDraggable() {
      this.$http({
        url: this.$http.adornUrl("/product/category/updateList"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message.success("菜单拖拽修改成功");
        this.getMenus();
        this.expandedKey = this.curCatPidListExpand;
        this.curCatPidListExpand = [];
      });
    },
    //拖拽成功完成时触发的事件
    handleDrop(draggingNode, dropNode, dropType, ev) {
      //主要从拖拽完成后的被指向节点(dropNode)中获取数据
      //0.需要确定拖拽完成后，被拖动节点的一下三种信息
      let curDragNodeChildNodes = []; //当前拖动节点的兄弟节点
      //1.确定当前被拖动节点的父Id
      if (dropType == "inner") {
        this.curCatPid = dropNode.data.catId;
        curDragNodeChildNodes = dropNode.childNodes;
        this.curCatPidListExpand.push(this.curCatPid);
      } else {
        this.curCatPid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        curDragNodeChildNodes = dropNode.parent.childNodes;
      }
      //2.确定当前被拖动节点的所有同级兄弟的排序
      if (curDragNodeChildNodes != null && curDragNodeChildNodes.length > 0) {
        for (let i = 0; i < curDragNodeChildNodes.length; i++) {
          let curBrotherNode = curDragNodeChildNodes[i];
          //如果此遍历节点是当前拖拽的节点，则进行第3步的逻辑处理
          if (curBrotherNode.data.catId == draggingNode.data.catId) {
            this.updateNodes.push({
              catId: curBrotherNode.data.catId,
              parentCid: this.curCatPid,
              sort: i,
              catLevel: curBrotherNode.level,
            });
            if (
              curBrotherNode.childNodes != null &&
              curBrotherNode.childNodes.length > 0
            ) {
              for (let i = 0; i < curBrotherNode.childNodes.length; i++) {
                this.updateNodes.push({
                  catId: curBrotherNode.childNodes[i].data.catId,
                  catLevel: curBrotherNode.level + 1,
                });
              }
            }
          }
          this.updateNodes.push({ catId: curBrotherNode.data.catId, sort: i });
        }
      }
      //3.确定被拖动节点的子节点的层级

      //这样应该会有重复的节点提交？？？
      console.log("将要更新的节点数据" + this.updateNodes);
    },
    nodeClick(data, node) {
    },
    //以最多三级分类为条件，判断是否能拖拽到指定位置
    allowDrop(draggingNode, dropNode, type) {
      this.getCurMaxLevel(draggingNode);
      let deep = this.maxLevel - draggingNode.level + 1; //要拖拽的数据的深度(包含它自身)
      if (type === "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.level <= 4;
      }
    },
    // 获取当前拖动元素的最深层级
    getCurMaxLevel(node) {
      this.maxLevel = node.level; //设置最大层级就是它自己,防止它没有子元素
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (this.maxLevel < node.childNodes[i].level) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.getCurMaxLevel(node.childNodes[i]);
        }
      }
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list"),
        method: "get",
      }).then(({ data }) => {
        //结构表达式{data},就相当于res.data
        this.menus = data.data;
      });
    },
    append(data) {
      this.dialogVisible = true;
      this.isEdit = "添加";
      this.dialogTitle = "添加";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //防止data.catLevel是字符串(需要的是数字)
      //数据重置(以访被编辑字段覆盖)
      this.category.catId = null;
      this.category.showStates = 1;
      this.category.name = "";
      this.category.sort = 0;
      this.category.productUnit = "";
      this.category.icon = "";
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
            this.$message.success("菜单删除成功");
            this.getMenus();
            //设置需要默认展开的菜单（被删除节点的父节点）
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    edit(data) {
      this.dialogVisible = true;
      this.isEdit = "修改";
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.name = data.data.name;
        this.category.showStatus = data.data.showStatus;
        this.category.parentCid = data.data.parentCid;
      });
    },
    handleCategory() {
      if (this.isEdit === "添加") {
        this.addCategory();
      }
      if (this.isEdit === "修改") {
        this.editCategory();
      }
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message.success("菜单新增成功");
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    editCategory() {
      //虽然没有传showStatus字段，但是mybatis的逻辑删除字段会默认作为sql语句中的where条件
      const { catId, icon, productUnit, name } = this.category;
      this.$http({
        url: this.$http.adornUrl(`/product/category/update`),
        method: "post",
        data: this.$http.adornData({ catId, icon, productUnit, name }, false),
      }).then(({ data }) => {
        this.$message.success("菜单修改成功");
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
  },
  //计算属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 -创建之前
  beforeMount() {}, //生命周期 -挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>

<style scoped>
</style>