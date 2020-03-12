<template>
  <transition name="fade">
    <div class="shelf-title" :class="{'hide-shadow': ifHideShadow}" v-show="shelfTitleVisible">
      <div class="shelf-title-text-wrapper">
        <span class="shelf-title-text">{{title}}</span>
        <span class="shelf-title-sub-text" v-show="isEditMode">{{selectedText}}</span>
      </div>
      <div class="shelf-title-btn-wrapper shelf-title-left" v-if="showClear">
        <span class="shelf-title-btn-text" @click="clearCache">{{$t('shelf.clearCache')}}</span>
      </div>
      <div class="shelf-title-btn-wrapper shelf-title-right" v-if="showEdit">
        <span class="shelf-title-btn-text"
              @click="onEditClick">{{isEditMode ? $t('shelf.cancel') : $t('shelf.edit')}}</span>
      </div>
      <!--返回按钮,进入分组的页面才会显示-->
      <div class="shelf-title-btn-wrapper shelf-title-left" v-if="showBack">
        <span class="icon-back" @click="back"></span>
      </div>
      <div class="shelf-title-btn-wrapper"
           :class="{'shelf-title-left': changeGroupLeft, 'shelf-title-right': changeGroupRight}" @click="changeGroup"
           v-if="showChangeGroup">
        <span class="shelf-title-btn-text">{{$t('shelf.editGroup')}}</span>
      </div>
    </div>
  </transition>
</template>

<script>
  import { storeShelfMixin } from '../../utils/mixin'
  import { clearLocalStorage, saveBookShelf } from '../../utils/localStorage'
  import { clearLocalForage } from '../../utils/localForage'

  export default {
    mixins: [storeShelfMixin],
    props: {
      title: String
    },
    computed: {
      emptyCategory() {
        return !this.shelfCategory || !this.shelfCategory.itemList || this.shelfCategory.itemList.length === 0
      },
      // 编辑按钮只有在书架列表页面下才显示,又或者在分组列表页面下但该分组中没有书籍才显示
      showEdit() {
        return this.currentType === 1 || !this.emptyCategory
      },
      // 清除缓存按钮,只有在书架列表页面下才显示
      showClear() {
        return this.currentType === 1
      },
      // 显示返回按钮
      showBack() {
        // 在分组列表页面且不在编辑模式下
        return this.currentType === 2 && !this.isEditMode
      },
      // 显示修改分组按钮
      showChangeGroup() {
        return this.currentType === 2 && (this.isEditMode || this.emptyCategory)
      },
      changeGroupLeft() {
        return !this.emptyCategory
      },
      changeGroupRight() {
        return this.emptyCategory
      },
      selectedText() {
        const selectedNumber = this.shelfSelected ? this.shelfSelected.length : 0
        return selectedNumber <= 0 ? this.$t('shelf.selectBook') : (selectedNumber === 1 ? this.$t('shelf.haveSelectedBook').replace('$1', selectedNumber) : this.$t('shelf.haveSelectedBooks').replace('$1', selectedNumber))
      },
      // 创建取消按钮
      popupCancelBtn() {
        return this.createPopupBtn(this.$t('shelf.cancel'), () => {
          this.hidePopup()
        })
      }
    },
    watch: {
      offsetY(offsetY) {
        if (offsetY > 0) {
          this.ifHideShadow = false
        } else {
          this.ifHideShadow = true
        }
      }
    },
    data() {
      return {
        ifHideShadow: true
      }
    },
    methods: {
      onComplete() {
        this.hidePopup()
        this.setShelfList(
                // 删除掉分组中的书籍
                this.shelfList.filter(book => book.id !== this.shelfCategory.id))
                .then(() => {
                  // 更新书架列表
                  saveBookShelf(this.shelfList)
                  this.$router.go(-1)
                  this.setIsEditMode(false)
                })
      },
      deleteGroup() {
        if (!this.emptyCategory) {
          // 当前分组有书
          this.setShelfSelected(this.shelfCategory.itemList)
          this.moveOutOfGroup(this.onComplete)
        } else {
          this.onComplete()
        }
      },
      changeGroupName() {
        this.hidePopup()
        // 弹出对话框
        this.dialog({
          showNewGroup: true,
          groupName: this.shelfCategory.title
        }).show()
      },
      hidePopup() {
        this.popupMenu.hide()
      },
      // 创建修改分组弹窗的按钮
      createPopupBtn(text, onClick, type = 'normal') {
        return {
          text: text,
          // type控制颜色: normal 灰色, danger 红色
          type: type,
          click: onClick
        }
      },
      showDeleteGroup() {
        this.hidePopup()
        // this.hidePopup()要执行200ms,所以下面的操作要在200ms之后进行
        setTimeout(() => {
          // 创建弹窗
          this.popupMenu = this.popup({
            // title: 删除分组后，分组内的书籍将会自动移出分组
            title: this.$t('shelf.deleteGroupTitle'),
            btn: [
              // 确认按钮,红色danger
              this.createPopupBtn(this.$t('shelf.confirm'), () => {
                this.deleteGroup()
              }, 'danger'),
              // 取消按钮
              this.popupCancelBtn
            ]
          }).show()
        }, 200)
      },
      // 修改分组按钮
      changeGroup() {
        this.popupMenu = this.popup({
          btn: [
            // 修改分组名按钮
            this.createPopupBtn(this.$t('shelf.editGroupName'), () => {
              this.changeGroupName()
            }),
            // 删除分组按钮
            this.createPopupBtn(this.$t('shelf.deleteGroup'), () => {
              this.showDeleteGroup()
            }, 'danger'),
            // 取消按钮
            this.popupCancelBtn
          ]
        }).show()
      },
      back() {
        this.$router.go(-1)
        this.setIsEditMode(false)
      },
      // 编辑按钮和取消按钮的切换
      onEditClick() {
        if (!this.isEditMode) {
          // 非编辑模式下,需要把被选中的图书清空,且需要把他们的选中状态给置为false
          this.setShelfSelected([])
          this.shelfList.forEach(item => {
            item.selected = false
            if (item.itemList) {
              // 如果是一个分组,则对这个分组下的每一本书的选中状态给置为false
              item.itemList.forEach(subItem => {
                subItem.selected = false
              })
            }
          })
        }
        this.setIsEditMode(!this.isEditMode)
      },
      clearCache() {
        clearLocalStorage()
        clearLocalForage()
        this.setShelfList([])
        this.setShelfSelected([])
        this.getShelfList()
        this.simpleToast(this.$t('shelf.clearCacheSuccess'))
      }
    }
  }
</script>

<style lang="scss" rel="stylesheet/scss" scoped>
  @import "../../assets/styles/global";

  .shelf-title {
    position: relative;
    z-index: 130;
    width: 100%;
    height: px2rem(42);
    background: white;
    box-shadow: 0 px2rem(2) px2rem(2) 0 rgba(0, 0, 0, .1);
    &.hide-shadow {
      box-shadow: none;
    }
    .shelf-title-text-wrapper {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: px2rem(42);
      @include columnCenter;
      .shelf-title-text {
        font-size: px2rem(16);
        line-height: px2rem(20);
        font-weight: bold;
        color: #333;
      }
      .shelf-title-sub-text {
        font-size: px2rem(10);
        color: #333;
      }
    }
    .shelf-title-btn-wrapper {
      position: absolute;
      top: 0;
      box-sizing: border-box;
      height: 100%;
      @include center;
      .shelf-title-btn-text {
        font-size: px2rem(14);
        color: #666;
      }
      .icon-back {
        font-size: px2rem(20);
        color: #666;
      }
      &.shelf-title-left {
        left: 0;
        padding-left: px2rem(15);
      }
      &.shelf-title-right {
        right: 0;
        padding-right: px2rem(15);
      }
    }
  }
</style>
