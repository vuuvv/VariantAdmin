<template>
  <!--
  <el-breadcrumb separator-class="el-icon-arrow-right">
    <el-breadcrumb-item :to="{ path: '/' }">系统设置</el-breadcrumb-item>
    <el-breadcrumb-item>实体管理</el-breadcrumb-item>
  </el-breadcrumb>
  -->

  <el-container>
    <el-header class="entity-action-section">实体列表
      <div style="float: right">
        <el-button type="primary" size="small" @click="createNewEntity">
          <i class="el-icon-plus"></i>&nbsp;新建实体</el-button>
      </div>
    </el-header>
    <el-main class="card-container">
      <el-card class="entity-card" shadow="hover" v-for="(entityItem, entityIdx) in entityItems" :key="entityIdx"
               @click.native=" (event) => showContextMenu(entityItem, event) "
               @contextmenu.native.prevent=" (event) => showContextMenu(entityItem, event) "
               @mouseenter.native="onEnterEntity(entityItem.name, entityItem.label, entityIdx)" @mouseleave.native="onLeaveEntity">
        <div slot="header">
          <span>{{entityItem.label}}</span>
        </div>
        <div>{{entityItem.name}}</div>
        <div v-if="!!entityItem.systemEntityFlag" class="system-flag system-entity"><i title="系统实体">系</i></div>
        <div v-if="!!!entityItem.detailEntityFlag" class="entity-flag main-entity"><i title="主实体">主</i></div>
        <div v-if="!!entityItem.detailEntityFlag" class="entity-flag detail-entity"><i title="明细实体">从</i></div>

        <!--
        <div v-if="(hoverEntityIdx === entityIdx) && !!!entityItem.detailEntityFlag" class="entity-item-tool">
          <el-button type="text" class="field-tool" icon="el-icon-menu"
                     @click="gotoEntityManager(entityItem.name, entityItem.label)">字段管理</el-button>
          <el-button type="text" class="layout-tool" icon="el-icon-s-operation"
                     @click="gotoFormLayout(entityItem.name, entityItem.label)">表单设计</el-button>
        </div>
        <div v-if="(hoverEntityIdx === entityIdx) && !!entityItem.detailEntityFlag" class="entity-item-tool">
          <el-button type="text" class="field-tool-center" icon="el-icon-menu"
                     @click="gotoEntityManager(entityItem.name, entityItem.label)">字段管理</el-button>
        </div>
        -->


      </el-card>

      <el-dialog title="新建实体" :visible.sync="showNewEntityDialogFlag" v-if="showNewEntityDialogFlag" :show-close="false"
            :close-on-click-modal="false" :close-on-press-escape="false">
        <EntityPropEditor ref="EPEditor" :entityProps="newEntityProps" :show-title="false"
                          :filter-entity-method="filterMainEntity"></EntityPropEditor>
        <div slot="footer" class="dialog-footer">
          <el-button type="primary" @click="saveNewEntity" size="small" style="width: 90px">保 存</el-button>
          <el-button @click="showNewEntityDialogFlag = false" size="small">取 消</el-button>
        </div>
      </el-dialog>

      <div id="contextMenu" v-show="contextMenuVisible" class="context-menu"
           @mouseenter="clearHideMenuTimer" @mouseleave="setHideMenuTimer">
        <div class="context-menu__item" @click="gotoEntityManager(selectedEntityObj)">字段管理</div>
        <div class="context-menu__item" @click="gotoFormLayout(selectedEntityObj)">表单设计</div>
        <div class="context-menu__item" @click="gotoListSetting(selectedEntityObj)">列表设计</div>
        <div class="context-menu__item" @click="deleteSelectedEntity(selectedEntityObj)">删除实体</div>
      </div>

    </el-main>
  </el-container>
</template>

<script>
  import {getEntitySet, createEntity, entityCanBeDeleted, deleteEntity} from '@/api/system-manager'
import EntityPropEditor from './entity-editor/entity-property-editor'

export default {
  name: 'EntityList',
  components: {EntityPropEditor},
  data () {
    return {
      entityItems: [],
      showNewEntityDialogFlag: false,
      newEntityProps: {
        'name': '',
        'label': '',
        entityCode: null,
        layoutable: true,
        listable: true,
        authorizable: true,
        assignable: false,
        shareable: false,
        mainEntity: '',
        detailEntityFlag: false,
      },

      hoverEntityIdx: -1,

      selectedEntityObj: null,
      contextMenuVisible: false,
      hideMenuTimerId: null,

    }
  },
  computed: {
    isLayoutable() {
      return !!this.selectedEntityObj && (this.selectedEntityObj.layoutable === true)
    },

    isListable() {

    },
  },
  mounted () {
    this.getEntityList()
  },
  methods: {
    getEntityList () {
      getEntitySet().then(res => {
        if (res.error != null) {
          this.$message({ message: res.error, type: 'error' })
          return
        }
        this.entityItems = res.data
      }).catch(res => {
        this.$message({ message: res.message, type: 'error' })
      })
    },

    createNewEntity() {
      this.newEntityProps.name = ''
      this.newEntityProps.label = ''
      this.newEntityProps.entityCode = null
      this.newEntityProps.layoutable = true
      this.newEntityProps.listable = true
      this.newEntityProps.authorizable = true
      this.newEntityProps.assignable = false
      this.newEntityProps.shareable = false
      this.newEntityProps.mainEntity = ''
      this.newEntityProps.detailEntityFlag = false
      this.showNewEntityDialogFlag = true
    },

    saveNewEntity() {
      if (!this.$refs['EPEditor'].validateForm())
        return

      // console.log(this.newEntityProps)
      let mainEntityName = !!!this.newEntityProps.mainEntity ? 'null' : this.newEntityProps.mainEntity
      createEntity(this.newEntityProps, mainEntityName).then(res => {
        if (res.error != null) {
          this.$message({ message: res.error, type: 'error' })
          return
        } else {
          this.$message.success('保存成功')
          this.showNewEntityDialogFlag = false
          this.getEntityList() // this.$emit('fieldSaved')
        }
      }).catch(res => {
        this.$message({ message: res.message, type: 'error' })
      })
    },

    onEnterEntity(entityName, entityLabel, entityIdx) {
      this.hoverEntityIdx = entityIdx
      // this.selectedEntityObj = {
      //   name: entityName,
      //   label: entityLabel
      // }
    },

    onLeaveEntity() {
      this.hoverEntityIdx = -1
      // this.selectedEntityObj = null
    },

    gotoEntityManager(selectedEntityObj) {
      //this.$router.push({name: 'EntityManager', params: {'entity': entityName, entityLabel}})
      this.$router.push({name: 'FieldManager',
        params: {
          entity: selectedEntityObj.name,
          entityLabel: selectedEntityObj.label,
        }
      })
    },

    gotoFormLayout(selectedEntityObj) {
      if (selectedEntityObj.layoutable !== true) {
        this.$message.info('当前实体不允许设计表单')
        return
      }

      this.$router.push({name: 'FormLayout',
        params: {
          'entity': selectedEntityObj.name,
          'entityLabel': selectedEntityObj.label
        }
      })
    },

    gotoListSetting(selectedEntityObj) {
      if (selectedEntityObj.listable !== true) {
        this.$message.info('当前实体不允许设计列表')
        return
      }

      this.$router.push({name: 'ListSetting',
        params: {
          'entity': selectedEntityObj.name,
        }
      })
    },

    deleteSelectedEntity(selectedEntityObj) {
      entityCanBeDeleted(selectedEntityObj.name).then(res => {
        if (res.error != null) {
          this.$message({message: res.error, type: 'error'})
          return
        }

        if (res.data !== true) {
          this.$message.info('提示：系统实体、内部实体不能被删除！')
          return
        }

        let confirmText = ['实体删除后不能恢复，是否确认删除?', '1. 删除实体会清空该实体的所有数据，包括实体所有字段、表单布局和列表设置，且不能恢复；’',
          '2. 如该实体包含明细实体，请先删除明细实体；', '3. 如有其他实体引用字段指向该实体，请手工删除引用字段；']
        const h = this.$createElement
        let pTags = []
        confirmText.forEach(ct => {
          pTags.push(h('p', null, ct))
        })
        this.$confirm('删除提醒', {
          message: h('div', null, pTags),
          showCancelButton: true,
          type: 'warning'
        }).then(() => {
          this.executeDeleteEntity(selectedEntityObj.name)
        }).catch(() => {
          this.$message({type: 'info', message: '已取消删除'});
        });
      }).catch(res => {
        this.$message({ message: res.message, type: 'error' })
      })
    },

    executeDeleteEntity(entity) {
      deleteEntity(entity).then(res => {
        if (res.error != null) {
          this.$message({ message: res.error, type: 'error' })
          return
        }

        this.$message.success('实体已删除')
        this.getEntityList()
      }).catch(res => {
        this.$message({ message: res.message, type: 'error' })
      })
    },

    filterMainEntity(filterList, callBack) {
      filterList.length = 0  /* 清空数组，不能用filterList=[]，否则SimpleTable显示不出数据！！ */
      filterList = this.entityItems.filter( entity => {
        if (entity.detailEntityFlag === false) {
          filterList.push({name: entity.name, label: entity.label})
        }
      })

      callBack()
    },

    hideContextMenu(event) {
      this.contextMenuVisible = false
      this.clearHideMenuTimer()
      //document.removeEventListener('click', this.hideContextMenu)
    },

    contextMenuSetting(menu, event) {
      if (event.clientX > 1800) {
        menu.style.left = event.clientX - 100 + 'px'
      } else {
        menu.style.left = event.clientX + 1 + 'px'
      }
      //document.addEventListener('click', this.hideContextMenu)
      if (event.clientY > 700) {
        menu.style.top = event.clientY - 30 + 'px'
      } else {
        menu.style.top = event.clientY - 10 + 'px'
      }
    },

    showContextMenu(entity, event) {
      this.clearHideMenuTimer()
      this.contextMenuVisible = false
      this.contextMenuVisible = true
      //event.preventDefault() //关闭浏览器右键默认事件
      this.selectedEntityObj = {
        name: entity.name,
        label: entity.label,
        layoutable: entity.layoutable,
        listable: entity.listable,
      }
      let menu = document.querySelector('#contextMenu')
      this.contextMenuSetting(menu, event)
      this.setHideMenuTimer()
    },

    setHideMenuTimer() {
      this.hideMenuTimerId = setTimeout(() => {
        this.hideContextMenu()
      }, 3000)
    },

    clearHideMenuTimer() {
      clearTimeout(this.hideMenuTimerId)
    },

  }
}
</script>

<style lang="scss" scoped>
  .el-main.card-container {
    background: #ffffff;
  }

  .el-main.card-container:after {
    content: '';
    display: block;
    clear: both;
  }

  .el-header.entity-action-section {
    height: 54px !important;  /* 解决内部按钮居中问题 */
    line-height: 54px;  /* 解决内部按钮居中问题 */
    font-size: 14px;
    text-align: center;
    border-bottom: 1px dashed #eeeeee;
  }

  .el-card.entity-card {
    font-size: 13px;
    width: 180px;
    float: left;
    margin: 10px;
    position: relative;
    cursor: pointer;

    ::v-deep .el-card__header {
      height: 42px;  /* 指定高度，以避免中英文字体高度不一致导致el-card自动换行出现问题 */
      text-align: center;
      border-bottom: 1px dotted #EBEEF5;
      padding: 12px;
    }

    ::v-deep .el-card__body {
      text-align: center;
      font-size: 14px;
      padding: 12px 12px 30px 12px;
    }

    .system-flag {
      position: absolute;
      top: 0;
      right: 23px;
      height: 22px;
      line-height: 22px;
      z-index: 9;

      i{
        font-size: 11px;
        color: #fff;
        margin: 0 5px;
        cursor: pointer;
      }
    }

    .system-flag.system-entity {
      background: #42b983;
    }

    .entity-flag {
      position: absolute;
      top: 0;
      right: 0;
      height: 22px;
      line-height: 22px;
      z-index: 9;

      i{
        font-size: 11px;
        color: #fff;
        margin: 0 5px;
        cursor: pointer;
      }
    }

    .entity-flag.main-entity {
      background: #4AB7BD;
    }

    .entity-flag.detail-entity {
      background: #cccccc;
    }

    .entity-item-tool {

      .field-tool {
        font-size: 12px;
        position: absolute;
        bottom: -5px;
        left: 6px;
      }

      .layout-tool {
        font-size: 12px;
        position: absolute;
        bottom: -5px;
        right: 6px;
      }

      .field-tool-center {
        font-size: 12px;
        position: absolute;
        bottom: -5px;
        left: 55px;
      }
    }
  }

  .context-menu {
    position: absolute;
    background-color: #fff;
    width: 100px;
    /*height: 106px;*/
    font-size: 12px;
    color: #444040;
    border-radius: 4px;
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    border-radius: 3px;
    border: 1px solid rgba(0, 0, 0, 0.15);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.175);
    white-space: nowrap;
    z-index: 1000;
  }

  .context-menu__item {
    display: block;
    line-height: 34px;
    text-align: center;
  }

  .context-menu__item:not(:last-child) {
    border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  }

  .context-menu__item.hidden {
    display: none;
  }

  .context-menu__item:hover {
    cursor: pointer;
    background: $--color-primary;
    border-color: #66b1ff;
    color: #fff;
  }

</style>
